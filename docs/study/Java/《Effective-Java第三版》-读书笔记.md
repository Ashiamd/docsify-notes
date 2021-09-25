# 《Effective-Java第三版》-读书笔记

> + `1、`、`2、`表示：第1章、第2章
>
> + `E1`、`E2`表示：第1条、第2条
> + `* X、`表示：个人认为第X章是重点（注意点）
> + `* EX`表示：个人认为第X条是重点（注意点）
>
> 全书共12章，90条目
>
> 下面提到的“设计模式”，指《Design Patterns - Elements of Reusable Object-Oriented Software》一书中提到的23种设计模式

# 1、引言

+ 本书中，API可以简单指：类、接口或者包。（API = Application Programming Interface）

  + 使用API编程的程序员，即该API的用户（user）
  + 在类实现中使用API的类被称为该API的客户端（client）

  - 类、接口、构造器、成员以及序列化形式统称为API元素（API element）
  - API由所有可在定义该API的包以外访问的API元素组成
  - 不严格地讲，一个包的导出API包括：
    - 包中的每个public类
    - 接口中所有public、protected成员和构造器

  - Java 9中新增模块系统（module system），如果类库中使用了模块系统，其API就是类库的模块声明导出的所有包的导出API组合

  ps：模块系统，这个学过前端技术的同学应该会更熟悉。

# 2、创建和销毁对象

+ 何时、如何创建（对象）；

+ 确保销毁（对象）、销毁前动作；

## E1 用静态工厂方法代替构造器

+ 概述

  提供一个公有的**静态工厂方法**（static factory method），返回类的实例。

  *ps：与设计模式中的工厂方法（Factory Method）无对应关系*

---

+ 举例

  1. Boolean（基本类型boolean的装箱类）

     ```java
     public static Boolean valueOf(boolean b) {
       return b ? Boolean.TRUE : Boolean.FALSE;
     }
     ```

---

+ 优势：

  1. （比起构造函数）方法名有意义，**可读性高**
  2.  每次调用**不一定创建新对象**
  3. **可以返回类型的任何子类型对象**
  4. **可通过调整传入的参数来改变返回的对象的类**
  5. **方法返回的对象所属的类，在编写包含该静态工厂方法的类时可以不存在**

  ---

  1. （比起构造函数）方法名有意义，**可读性高**

     + 构造函数仅通过重载实现多种类的实例，仅通过入参区分版本，可读性低，容易导致误解。
     + 需要多个带有相同签名的构造器（入参相同，仅传参顺序不同）时，考虑使用**方法名有意义**的静态工厂方法代替构造函数。

  2.  每次调用**不一定创建新对象**

     + （E17）返回预先创建的实例 or **缓存实例重复使用**。

     + `Boolean.valueOf(boolean)` 典例，无需创建新对象。

     + 类似享元模式（Flyweight），线程池技术就是典型的享元模式。

     + 严控实例创建时机，被称为"实例受控的类（instance-controlled）"。实例受控的类特性：

       + 可用于实现Singleton（单例类）（E3）
       + 使得类不可实例化（E4）
       + 可用于确保不可变的值类不会存在两个"相同实例"，实现当且仅当a\==b时，a.equals(b)\==true

       以上几点同时也是实现享元模式的基础。

       枚举（enum）类型保证了以上几点。（ps：枚举是最好的单例实现方式）

  3. **可以返回类型的任何子类型对象**

     + **灵活应用：API返回某对象，同时该对象又不需要是公有的。该技巧适用于基于接口的框架（interface based framework）**（E20）

     + 良好习惯：可要求客户端通过接口（而非其实现类）来引用被返回的对象（E64）

     > **在Java 8 之前，接口不能有静态方法**，因此按照惯例，接口 Type 的静态工厂方法被放在一个名为Types 的不可实例化的伴生类（E4）中。例如Java Collections Framework的集合接口有45 个工具实现，分别提供了不可修改的集合、同步集合，等等。几乎所有这些实现都通过静态工厂方法在一个不可实例化的类（java.util.Collections）中导出。所有返回对象的类都是非公有的。
     >
     > Java8中，接口支持静态方法，无需给接口再提供伴生类，原先的静态方法可单独封装到一个包级私有的类中（因为**Java8仍要求接口的所有静态方法必须是公有的**）。
     >
     > **Java 9 中允许接口有私有的静态方法，但是静态域和静态成员类仍然需要是公有的**。

  4. **可通过调整传入的参数来改变返回的对象的类**

     + 只要是已声明的**返回类型的子类型**，都是允许的。返回对象的类也可能随着发行版本的不同而不同。
     + EnumSet（E36）没有公有的构造器，只有静态工厂方法。

     > EnumSet中的`noneOf`静态方法，根据传入的底层枚举类型的大小，返回`RegularEnumSet`和`JumboEnumSet`这两种子类其中一种的实例。
     >
     > + 枚举类型内含有原素<=64个，返回`RegularEnumSet`实例，用单个long进行支持
     > + 枚举类型内含有原素>=65个，返回`JumboEnumSet`实例，用一个long数组进行支持
     >
     > 这两个实现类的存在对于客户端不可见（非public的class）。
     >
     > 如果RegularEnumSet 不能再给小的枚举类型提供性能优势，就可能从未来的发行版本中将它删除，不会造成任何负面的影响。同样地，如果事实证明对性能有好处，也可能在未来的发行版本中添加第三甚至第四个EnumSet 实现。<u>客户端永远不知道也不关心它们从工厂方法中得到的对象的类，它们只关心它是EnumSet 的某个子类</u>。
     >
     > ```java
     > /**
     >      * Creates an empty enum set with the specified element type.
     >      *
     >      * @param <E> The class of the elements in the set
     >      * @param elementType the class object of the element type for this enum
     >      *     set
     >      * @return An empty enum set of the specified type.
     >      * @throws NullPointerException if {@code elementType} is null
     >      */
     > public static <E extends Enum<E>> EnumSet<E> noneOf(Class<E> elementType) {
     >   Enum<?>[] universe = getUniverse(elementType);
     >   if (universe == null)
     >     throw new ClassCastException(elementType + " not an enum");
     > 
     >   if (universe.length <= 64)
     >     return new RegularEnumSet<>(elementType, universe);
     >   else
     >     return new JumboEnumSet<>(elementType, universe);
     > }
     > ```

  5. **方法返回的对象所属的类，在编写包含该静态工厂方法的类时可以不存在**

     + 该灵活性构成了服务提供者框架（Service Provider Framework）的基础，例如JDBC（Java数据库连接）API。

     > 服务提供者框架：多个服务提供者实现一个服务，系统为服务提供者的客户端提供多个实现，并把它们从多个实现中解稠出。
     >
     > 服务提供者框架包含3重要组件，1可选组件：
     >
     > + 服务接口（Service Interface）：由提供者实现
     > + 提供者注册API（Provider Registration API）：提供者注册实现的途径
     > + 服务访问API （Service Access API）：用于客户端获取服务实例。服务访问API 是客户端用来指定某种选择实现的条件。如果没有这样的规定， API 就会返回默认实现的一个实例，或者允许客户端遍历所有可用的实现。服务访问API 是"灵活的静态工厂"，它构成了服务提供者框架的基础。
     > + 服务提供者接口（Service Provider Interface）-可选：它表示产生服务接口之实例的工厂对象。如果没有服务提供者接口，实现就通过**反射**方式进行实例化（E65）。
     >
     > ---
     >
     > 对于JDBC来说， Connection接口就是其"服务接口"的一部分，DriverManager.registerDriver是"提供者注册API"，DriverManager.getConnection是"服务访问API"，Driver是"服务提供者接口"。
     >
     > ---
     >
     > 服务提供者框架变种多，例如：
     >
     > + 桥接模式（Bridge），"服务访问API"返回’比提供者需要的‘更为丰富的服务接口。
     > + 依赖注入框架（E5）
     > + **Java6起，提供了通用的"服务提供者框架" - `java.util.ServiceLoader`**（E59）
     >
     > JDBC出现更早，所以没使用`java.util.ServiceLoader`实现。
     >
     > ---
     >
     > + [java.util.ServiceLoader使用_石头的专栏-CSDN博客_java.util.serviceloader](https://blog.csdn.net/kokojhuang/article/details/8273303)
     >
     > + [java.util.ServiceLoader加载服务实现类 - 简书 (jianshu.com)](https://www.jianshu.com/p/7d0caf0f9d3f)
     > + [Java SPI机制：ServiceLoader实现原理及应用剖析 (juejin.cn)](https://juejin.cn/post/6844903891746684941) <= 较详细，有时间的话推荐阅读

+ 劣势：

  1. 类如果不含public的或者protected的构造器，就不能被子类实例化
  2. 较难被发现

  ---

  1. 类如果不含public的或者protected的构造器，就不能被子类实例化

     但也因此鼓励使用复合（composition）代替继承（E18），这是不可变类型所需要的（E17）。

     *ps：compostion这个概念可以看看设计模式，如果从代码层面看，大致就是将类型B的对象作为类型A的成员变量，并且B随着A实例化时实例化，随着A销毁时销毁，B是A不可分割的一部分。另一个类似的概念是Aggregation。*

  2. 较难被发现

     需要程序员自己找到类中用于实例化的方法。（方法多的时候可能就不清楚用哪个）

     可以通过遵循标准的命名习惯来弥补该劣势。

     以下是静态工厂方法的一些惯用名称，其中一部分如下：

     1. from——类型转换方法，只有单个参数，返回该类型的一个相对应的实例，例如：

        `Date d = Date.from(instant);`

     2. of——聚合方法，带有多个参数，返回该类型的一个实例，把他们合并起来，例如：

        `Set<Rank> faceCards = EnumSet.of(JACK, QUEEN, KING);`

     3. valueOf——比from和of更繁琐的一种替代方法，例如：

        `BigInteger prime = BigInteger.valueOf(Integer.MAX_VALUE);`

     4. instance或者getInstance——返回的实例是通过方法的（如有）参数来描述的，但是不能说与参数具有同样的值，例如：

        `StackWalker luke = StackWalker.getInstance(options);`

     5. create或者newInstance——像instance或者getInstance一样，但create或者newInstance能够确保每次调用都返回一个新的实例，例如：

        `Object newArray = Array.newInstance(classObject, arrayLen);`

     6. getType——像getInstance一样，但是在工厂方法处于不同的类中的时候使用。Type表示工厂方法所返回的对象类型，例如：

        `FileStore fs = Files.getFileStore(path);`

     7. newType——像newInstance一样，但是在工厂方法处于不同的类中的时候使用。

        Type表示工厂方法所返回的对象类型，例如：

        `BufferedReader br = Files.newBufferedReader(path);`

     8. type——getType和newType的简版，例如：

        `List<Complaint> litany = Collections.list(legacyListany);`

---

- 结语：

  静态工厂方法和公有构造器各有千秋，创建实例时，切忌第一反应就是共有工造器，也可考虑静态工厂。

## E2 遇到多个构造器参数时要考虑使用构建器

- 概述：

  静态工厂和构造器有个共同的局限性：<u>它们都不能很好地扩展到大量的可选参数</u>。

  对于这类场景，一般常见实用重叠构造器（telescoping constructor）模式，该模式下，提供第一个构造器只有必要的参数，第二个构造器有一个可选参数，第三个有两个可选参数，一次类推，最后一个构造器包含所有可选的参数。

- 举例：

  + 重叠构造器模式（telescoping constructor）

    + 优势：参数统一校验，安全性高
    + 劣势：参数足够多时，可读性差

  + JavaBeans模式

    + 优势：可读性高
    + 劣势：
      + 因为构造过程被分到了几个调用中，**在构造过程中Java Bean可能处于不一致的状态**
      + **Java Beans模式使得把类做成不可变的可能性不复存在**（E17），这就需要程序员付出额外的努力来确保它的线程安全。

  + 建造者模式（Builder）

    + 优势：
      + **可读性高，Builder模式模拟了具名的可选参数**
      + **可适用于类层次结构**

    + 劣势：创建对象前，必须先创建构造器。（十分注重性能的情况下可能成问题）

  ​	**它（Builder）既能保证像重叠构造器模式那样的安全性，也能保证像Java Beans 模式那么好的可读性**。

  > 简而言之， **如果类的构造器或者静态工厂中具有多个参数，设计这种类时， Builder模式就是一种不错的选择**， 特别是当大多数参数都是可选或者类型相同的时候。与使用重叠构造器模式相比，使用Builder模式的客户端代码将更易于阅读和编写，构建器也比JavaBeans更加安全。
  >
  > 关于Builder的关键字还有：
  >
  > + 递归类型参数（recursive type parameter）
  > + 模拟的self模型（simulated self-type）
  > + 协变返回类型（covariant return type）

  ---

  - 重叠构造器模式（telescoping constructor）

    ```java
    // Telescoping constructor pattern - does not scale well!
    public class NutritionFacts {
      private final int servingSize; // (mL)	required
      private final int servings;		// (per container)	required
      private final int calories;		// (g/serving)		optional
      private final int fat;				// (mg/serving)		optional
      private final int carbohydrate	// (g/serving)	optional
    
        public NutritionFacts(int servingSize, int servings) {
        this(servingSize, servings, 0);
      }
    
      public NutritionFacts(int servingSize, int servings, int calories) {
        this(servingSize, servings, calories, 0);
      }
    
      public NutritionFacts(int servingSize, int servings, int calories, int fat) {
        this(servingSize, servings, calories, fat, 0);
      }
    
      public NutritionFacts(int servingSize, int servings, int calories, int fat, int sodium) {
        this(servingSize, servings, calories, fat, sodium, 0);
      }
    
      public NutritionFacts(int servingSize, int servings, int calories, int fat, int sodium, int carbohydrate) {
        this.servingSize = servingSize;
        this.servings = servings;
        this.calories = calories;
        this.fat = fat;
        this.sodium = sodium;
        this.carbohydrate = carbohydrate;
      }
    }
    ```

    ​	**简而言之，重叠构造器模式可行，但是当有许多参数的时候，客户端代码会很难缩写，并且仍然较难以阅读**。

    ​	如果读者想知道那些值是什么意思，必须很仔细地数着这些参数来探个究竟。一长串类型相同的参数会导致一些微妙的错误。如果客户端不小心颠倒了其中两个参数的顺序，编译器也不会出错，但是程序在运行时会出现错误的行为（E51） 。

    ---

  - JavaBeans模式

    先调用无参构造函数创建对象，再带哦用setter方法来设置参数。

    优势：

    + 可读性高

    劣势：

    + 因为构造过程被分到了几个调用中，**在构造过程中Java Bean可能处于不一致的状态**
    + **Java Beans模式使得把类做成不可变的可能性不复存在**（E17），这就需要程序员付出额外的努力来确保它的线程安全。

    > [Object.freeze(); 方法冻结一个对象。 - 秋风2016 - 博客园 (cnblogs.com)](https://www.cnblogs.com/lml2017/p/10435620.html) <= JS里的。书中提到可通过freeze方法来保证java bean不被修改，我本地java没找到这个方法，可能早期jdk版本有？或者说可自行实现freeze方法。

    ---

  - 建造者模式（Builder）

    ​	它不直接生成想要的对象，而是让客户端利用所有必要的参数调用构造器（或者静态工厂），得到一个builder对象。然后客户端在builder对象上调用类似于setter的方法，来设置每个相关的可选参数。最后客户端调用无参的build方法来生成通常是不可变的对象。这个buiider通常是它构建的类的静态成员类（E24） 。

    ```java
    // Builder Pattern
    public class NutritionFacts {
      private final int servingSize;
      private final int servings;
      private final int calories;
      private final int fat;
      private final int sodium;
      private final int carbohydrate;
      
      public static class Builder {
        // Required parameters
        private final int servingSize;
        private final int servings;
        
        // Optional parameter - initialized to default values
        private int calories = 0;
        private int fat = 0;
        private int sodium = 0;
        private int carbohydrate = 0;
        
        public Builder(int servingSize, int servings) {
          this.servingSize = servingSize;
          this.servings = servings;
        }
        
        public Builder calories(int val) { calories = val; return this; }
        public Builder fat(int val) { fat = val; return this; }
        public Builder sodium(int val) { sodium = val; return this; }
        public Builder carbohydrate(int val) { carbohydrate = va; return this; }
        
        public NutritionFacts build() {
          return new NutritionFacts(this);
        }
        
        private NutritionFacts(Builder builder) {
          servingSize = builder.servingSize;
          servings = builder.servings;
          calories = builder.calories;
          fat = builder.fat;
          sodium = builder.sodium;
          carbohydrate = builder.carbohydrate;
        }
      }
    }
    ```

    ​	注意NutritionFacts是**不可变**的，所有的默认参数值都单独放在一个地方。builder的设值方法返回builder本身，以便把调用链接起来，得到一个流式的API 。下面就是其客户端代码：

    `NutritionFacts cocaCola = new NutritionFacts.Builder(240, 8).calories(100).sodium(35).carbohydrate(27).build();`

    ​	**可读性高，Builder模式模拟了具名的可选参数，就像Python和Scala编程语言中的一样**。

    ​	为了简洁起见，示例中省略了有效性检查。要想尽快侦测到无效的参数， 可以在builder的构造器和方法中检查参数的有效性。查看不可变量，包括build方法调用的构造器中的多个参数。为了确保这些不变量免受攻击， 从builder 复制完参数之后，要检查对象域（E50） 。如果检查失败，就抛出IllegalArgumentException（E72），其中的详细信息会说明哪些参数是无效的（E75） 。

    ​	**Builder 模式也适用于类层次结构**。

    ​	使用平行层次结构的builder 时， 各自嵌套在相应的类中。抽象类有抽象的builder ，具体类有具体的builder。假设用类层次根部的一个抽象类表示各式各样的披萨：

    ```java
    // Builder pattern for class hierarchies
    public abstract class Pizza {
      public enum Topping { HAM, HUSHROOM, ONION, PEPPER, SAUSAGE }
      final Set<Topping> toppings;
      
      abstract static class Builder<T extends Builder<T>> {
        EnumSet<Topping> toppings = EnumSet.noneOf(Topping.class);
        public T addTopping(Topping topping) {
          toppings.add(Objects.requireNonNull(topping));
          return self();
        }
        abstract Pizza build();
        // Subclasses must override this method to return "this" protected abstract T self();
        protected abstract T self();
      }
      Pizza(Builder<?> builder) {
        toppings = builder.toppings.clone(); // See Item 50
      }
    } 
    ```

    ​	注意，Pizza.Builder的类型是**泛型**（generic type），带有一个**递归类型参数（recursive type parameter ）**，（E30）。它和抽象的self 方法一样，允许在子类中适当地进行方法链接，不需要转换类型。这个针对Java缺乏self类型的解决方案，被称作**模拟的self类型（simulated self-type）**。

    > [Java泛型 自限定类型(Self-Bound Types)详解_anlian523的博客-CSDN博客](https://blog.csdn.net/anlian523/article/details/102511783)

    ​	这里有两个具体的Pizza 子类，其中一个表示经典纽约风味的比萨，另一个表示馅料内置的半月型（ calzone）比萨。前者需要一个尺寸参数，后者则要你指定酱汁应该内置还是外置：

    ```java
    public class NyPizza extends Pizza {
      public enum Size { SMALL, MEDIUM, LARGE }
      private final Size size;
      
      public static class Builder extends Pizza.Builder<Builder> {
        private final Size size;
        
        public Builder(Size size) {
          this.size = Objects.requireNonNull(size);
        }
        @Override public NyPizza build() { return new NyPizza(this); }
        @Override protected Builder self() { return this; }
      }
      
      private NyPizza(Builder builder) {
        super(builder);
        size = builder.size;
      }
    }
    ```

    ```java
    public class Calzone extends Pizza {
      private final boolean sauceInside;
      
      public static class Builder extends Pizza.Builder<Builder> {
        private boolean sauceInside = false; // Default
        
        public Buider sauceInside() {
          sauceInside = true;
          return this;
        }
        
        @Override public Calzone build() { return new Calzone(this); }
        @Override protected Builder self() { return this; }
        
        private Calzone(Builder builder) {
          super(builder);
          sauceInside = builder.sauceInside;
        }
      }
    }
    ```

    ​	注意，每个子类的构造器中的build方法，都声明返回各自的子类：NyPizza.Builder的build方法返回NyPizza，而Calzone.Builder中的则返回Calzone。在该方法中，子类方法声明返回父类中声明的返回类型的子类型，这被称为**协变返回类型（covariant return type）**。<u>它允许客户端无须转换类型就能使用这些构造器</u>。

    > [java协变返回类型_Java中的协变返回类型_cumt951045的博客-CSDN博客](https://blog.csdn.net/cumt951045/article/details/107792592)

    ​	这些"层次化构建器"的客户端代码本质上与简单的NutritionFacts构建器一样。为了简洁起见，下列客户端代码示例是在枚举常量上静态导入：

    ```java
    NyPizza pizza = new NyPizza.Builder(SMALL)
      .addTopping(SAUSAGE).addTopping(ONION).build();
    Calzone calzone = new Calzone.Builder()
      .addTopping(HAM).sauceInside().build();
    ```

    ​	与构造器相比， builder 的微略优势在于，它**可以有多个可变（ varargs ） 参数**。因为builder 是利用单独的方法来设置每一个参数。此外，构造器还可以将多次调用某一个方法而传人的参数集中到一个域中，如前面的调用了两次addTopping 方法的代码所示。

    ​	Builder不足之处，创建对象前，必须先创建构造器。（十分注重性能的情况下可能成问题）

    ​	<u>Builder 模式还比重叠构造器模式更加冗长，因此它只在有很多参数的时候才使用，比如4 个或者更多个参数</u>。

    > 但是记住，将来你可能需要添加参数。如果一开始就使用构造器或者静态工厂，等到类需要多个参数时才添加构造器，就会无法控制，那些过时的构造器或者静态工厂显得十分不协调。因此，通常最好一开始就使用构建器。

    ​	简而言之， **如果类的构造器或者静态工厂中具有多个参数，设计这种类时， Builder模式就是一种不错的选择**， 特别是当大多数参数都是可选或者类型相同的时候。与使用重叠构造器模式相比，使用Builder模式的客户端代码将更易于阅读和编写，构建器也比JavaBeans更加安全。

## E3 用私有构造器或者枚举类型强化Singleton 属性

+ 概述

  Singleton通常用来代表一个无状态的对象，如函数（E24），或者本质唯一的系统组件。

+ 举例

  实现Singleton有两种常见的方式。其共同点：

  + 构造器私有
  + 导出公有静态成员（以便客户端能够访问该类的唯一实例）

  ---

  + 方式一：

    **公有静态成员是一个final域**

    ```java
    // Singleton with public final field
    public class Elvis {
      public static final Elvis INSTANCE = new Elvis();
      private Elvis() {...}
      public void leaveTheBuilding(){...}
    }
    ```

    + **私有构造器仅被调用一次**，用来实例化静态final域Elvis.INSTANCE。由于缺少公有或者受保护的构造器，保证了Elvis的全局唯一性。

    + **客户端可以借助`AccessibleObject.setAccessible`方法，通过反射机制（E65）调用私有构造器**。

      （如果要抵御这种攻击，可以修改构造器，在其被哟求创建第二个实例时抛出异常）

  + 方式二：

    **公有的成员是个静态工厂方法**

    ```java
    // Singleton with static factory
    public class Elvis {
      private static final Elvis INSTANCE = new Elvis();
      private Elvis() {...}
      public static Elvis getInstance() { return INSTANCE; }
      public void leaveTheBuilding() { ... }
    }
    ```

    公有域方法的主要优势：

    + API很清楚地表明这个类是一个Singleton：公有的静态域是final的，所以该域总是包含相同的对戏那个引用。
    + 实现简单

    + 灵活性高（可以改写成泛型Singleton工厂（E30））
    + 可以通过方法引用（method reference）作为提供者，比如`Elvis::instance`就是一个`Supplier<Elvis>`

    除非满足以上任意一种优势，否则应该优先考虑公有域（public-field）的方法。

  ---

  ​	为了将利用上述方法实现的Singleton类变成可序列化的（Serializable）（12、），仅在声明中加上`implements Serializable`是不够的。

  ​	**为了维护并保证Singleton，必须声明所有实例域都是transient的，并提供一个readResolve方法（E89）**。否则，每次反序列化一个序列化的实例，都会创建一个新的实例。

  ​	在Elvis类中加入readResolve方法如下：

  ```java
  // readResolve method to preserve singleton property
  private Object readResolve() {
    // Return the one true Elvis and let the garbage collector
    // take care of the Elvis impersonator
    return INSTANCE;
  }
  ```

  ---

  + 方式三：

  **声明一个包含单个元素的枚举类型**

  ```java
  // Enum singleton - the preferred approach
  public enum Elvis {
    INSTANCE;
    
    public void leaveTheBuilding() {...}
  }
  ```

  ​	这种方法在功能上与公有域方法类似，但更加简洁，**无偿提供了序列化机制，绝对防止多次实例化**，即时是在面对复杂的序列化或者反射攻击的时候。

  + **单元素的枚举类型经常成为实现Singleton的最佳方法**

  > 如果Singleton必须扩展一个超类，而不是扩展Enum的时候，则不宜使用该方法（虽然可以声明枚举去实现接口）。

## E4 通过私有构造器强化不可实例化的能力

+ 概述

  **不应该为了使某类不被实例化而特地声明为抽象类**。这样容易误导用户该类是为了继承而设计的。（E19）

  推荐的做法：**让这个类包含一个私有构造器，这样它就不能被实例化**

  *ps：因为只有当类不包含显式的构造器时，编译器才会生成缺省的构造器*

+ 举例

  ```java
  // Noninstantiable utility class
  public class UtilityClass {
    // Suppress default constructor for nonistantibility
    private UtilityClass() {
      throw new AssertionError();
    }
    ... // Remainder omitted
  }
  ```

  AssertionError非必需，但是可以避免不小心在类内部调用构造器。

+ 副作用

  **不能被子类化**

  ps：所有的构造器都必须显式或隐式调用超类（superclass）构造器。这种情况下子类就没有可访问的超类构造器可调用了。

## E5 优先考虑依赖注人来引用资源

- 概述

  **静态工具类和Singleton类不适合于需要引用底层资源的类**。

  ---

- 举例

  一个类需要能够创建多实例，每个实例都是用客户端指定的资源。满足该需求的简单的模式是：

  **当创建一个新的实例时，就将该资源传到构造器中**。

  ```java
  // Dependency injection provides flexibility and testability
  public class SpellChecker {
    private final Lexicon dictionary;
    
    public SpellChecker(Lexicon dictionary) {
      this.dictionary = Objects.requireNonNull(dictionary);
    }
    
    public boolean isValid(Stirng word) {...}
   	public List<String> suggestions(String typo) {...}
  }
  ```

  这是依赖注入（dependency injection）的一种形式

  ---

- 特性

  依赖注入的对象资源具有不可变性（E17），因此多个客户端可以共享依赖对象（假设客户端们想要的是同一个底层资源）。

  依赖注入也同样适用于构造器、静态工厂（E1）和构造器（E2）

---

- 变体

  **将资源工厂（factory）传给构造器。**

  工厂具体表现为工厂方法（Factory Method）。**在Java8中增加的接口Supplier\<T\>最适合用于表示工厂**。

  ---

- 变体举例

  带有Supplier\<T>的方法，通常应该限制输入工厂的类型参数使用*有限制的通配符类型*（bounded wildcard type）（E31）。

  ```java
  Mosaic create(Supplier<? extends Tile> tileFactory) {...}
  ```

---

- 依赖注入的优点

  提高灵活性和可测试性

- 依赖注入的缺点

  大项目通常包含上千个依赖，管理时凌乱不堪。

  此时应该考虑使用依赖注入框架（dependency injection framework），如Dagger、Guice、Spring。

  > **设计成手动依赖注入的API，一般都适用于这些框架**

---

+ 小结

  ​	不要使用SIngleton和静态工具类来实现依赖一个或多个底层资源的类；也不要直接使用这个类来创建这些资源。

  ​	**将资源或工厂传给构造器（或者静态工厂，或者构造器），通过它们来创建类**。

  ​	这种实线就成为依赖注入，提高类的灵活性、可重用性和可测试性。

## * E6 避免创建不必要的对象

+ 概述

  尽可能复用单个对象。如果对象是不可变的（immutable）（E17），它就始终可以被重用。

+ 举例

  + 错误使用方式

    ```java
    String s = new String("test");
    ```

    每次都会新建一个新的String实例

  + 正确方式

    ```java
    String s = "test";
    ```

    该版本只用了一个String实例。

    **对于同一个虚拟机中运行的代码，只要它们包含相同的字符串字面常量，该对象就会被重用**。

---

- 扩展

  ​	对于同时提供了静态工厂方法（static factory method）（E1）和构造器的不可变类，**通常优先使用静态工厂方法而不是构造器，以避免创建不必要的对象**。

  ​	有些对象创建成本比其他对象要高得多，这类对象建议缓存下来重用。例如正则表达式中的Pattern实例。

  ```java
  // Performance can be greatly improved!
  static boolean isRomanNumeral(String s) {
    return s.matches("^(?=.)M*(C[MD]|D?C{0,3})(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");
  }
  ```

  ​	上述实现中，**`String.matches`方法最易于查看一个字符串是否于正则表达式相匹配，但并不适合在注重性能的情形中重复使用**。

  ​	它在内部为正则表达式创建了一个Pattern实例，但只用了一次，之后就可以进行垃圾回收了。**创建Pattern实例的成本很高，因为需要将正则表达式编译成一个有限状态机（finite state machine）**。

  ​	为了提升性能，应该显式地将正则表达式编译成一个Pattern实例（不可变），让它成为类初始化的一部分，并将它缓存起来，每当调用isRomanNumeral方法的时候就重用同一个实例：

  ```java
  // Reusing expensive object for improved performance
  public class RomanNumerals {
    private static final Pattern ROMAN = Pattern.compile("^(?=.)M*(C[MD]|D?C{0,3})(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");
    
    static boolean isRomanNumeral(String s){
      return ROMAN.matcher(s).matches();
    }
  }
  ```

  ​	*改进后的isRomanNumeral方法如果被频繁调用，会显示出明显的性能优势。在作者的机器上，速度提高了6.5倍（都是us级别）。同时，提取Pattern实例用final修饰时，可以起名，增加可读性。*

---

- 自动装箱（autoboxing）

  ```java
  private static long sum(){
   	Long sum = 0L;
    for(long i = 0; i <= Integer.MAX_VALUE; ++i){
      sum +=i;
    }
    return sum;
  }
  ```

  上面sum声明为Long，每次增加long时构造一个实例，程序大约构造了2<sup>31</sup>个多余的Long实例。

  将sum从Long改成long，在作者机器上从6.3s减少到0.59秒。（我这里是6.041s降低到0.562秒）。

  结论：**优先使用基本类型而不是装箱基本类型，当心无意识的自动装箱**。

---

+ 对象池

  ​	正确使用对象池的典型对象示例就是数据库连接池。建立数据库连接的代价是非常昂贵的，因此重用这些对象非常有意义。

---

- 保护性拷贝

  + 当你应该重用现有对象的时候，请不要创建新的对象
  + 当你应该创建新对象的时候，请不要重用现有的对象（E50）

  ​	必要时如果没能实施保护性拷贝，将会导致潜在的Bug和安全漏洞；而不必要地创建对象则会影响程序的风格和性能。

## * E7 消除过期的对象引用

+ 举例

  ```java
  // Can you spot the "memory leak"
  public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;
  
    public Stack() {
      elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }
  
    public void push(Object e){
      ensureCapacity();
      elements[size++] = e;
    }
  
    public Object pop() {
      if(size == 0)
        throw new EmptyStackException();
      return elements[--size];
    }
  
    /**
     * Ensure space for at least one more elemnt, roughly
     * doubling the capacity each time the array needs to grow.
     */
    private void ensureCapacity() {
      if(elements.length == size)
        elements = Arrays.copyOf(elements, 2 * size + 1);
    }
  }
  ```

  ​	泛型版本参考（E29）。该程序存在隐藏的"内存泄漏"，随着内存占用不断增加，极端情况下会导致**磁盘交换**（Disk Paging），甚至导致程序失败（OutOfMemoryError错误），尽管这种失败情形相对少见。

  ---

  分析：

  ​	程序中发生内存泄漏的原因：**如果一个栈先是增长，然后收缩，那么从栈中弹出来的对象将不会被当作垃圾回收，即使使用栈的程序不再引用这些对象，它们也不会被回收**。

  + **栈内部维护这些对象的过期引用（obsolete reference）——永远也不会再被解除的引用**。

  本例中，凡是在elements数组的“活动部分”（active portion）以外的任何引用都是过期的。

  *ps：活动部分指elements中下标小于size的那些元素*

---

+ 处理"无意识的对象保护"（unintentional object retention）

  一旦一个对象引用已经过期，只需清空这个引用即可。对于上述的示例，修改pop方法如下：

  ```java
  public Object pop() {
    if(size == 0)
      throw new EmptyStackException();
    Object result = elements[--size];
    elements[size] = null; // Eliminate obsolete reference
    return result;
  }
  ```

---

+ **清空对象引用应该是一种例外，而不是一种规范行为**。

  那么，何时应该清空引用呢？Stack类的哪方面特性使它易于遭受内存泄漏的影响呢？

  ​	简而言之，问题在于，Stack类自己管理内存。存储池（storage pool）包含了elements数组（**对象引用**单元，而不是对象本身）的元素。

  ​	**数组活动区域（同前面的定义）中的元素是已分配的（allocated），而数组其余部分的元素则是自由的（free）。但是垃圾回收器并不知道这一点；对于垃圾回收器而言，elements数组中的所有对象引用都同等有效**。只有程序员知道数组的非活动部分是不重要的。程序员可以把这个情况告知垃圾回收器，做法很简单旦数组元素变成了非活动部分的一部分，程序员就手工清空这些数组元素（赋值null）。

---

+ **只要类是自己管理内存，就应该警惕内存泄漏问题**。一旦元素被释放掉，则该元素中包含的任何对象引用都应该被清空。

---

- **内存泄露的另一个常见来源：缓存**

  对象引用放到缓存中，长时间不用容易被遗忘，但是又仍然存在于缓存中。

  如果正好要实现这样的缓存：只要在缓存之外存在对某个项的键的引用，该项就有意义。那么可以使用WeakHashMap代表缓存。当缓存中的项过期之后，它们就会被自动删除。

  **只有当所要的缓存项的生命周期是由该键的外部引用而不是由值决定时，WeakHashMap才有用处**。

  > **更为常见的情形则是，"缓存项的生命周期是否有意义"并不是很容易确定，随着时间的推移，其中的项会变得越来越没有价值**。在这种情况下，缓存应该时不时地清除掉没用的项。这项清除工作可以由一个后台线程（可能是 ScheduledThreadPoolExecutor）来完成，或者也可以在给缓存添加新条目的时候顺便进行清理。 **LinkedHashMap**类利用它的removeEldestEntry方法可以很容易地实现后一种方案。对于更加复杂的缓存，必须直接使用**java.lang.ref**

  > [WeakHashMap的详细理解_qiuhao9527的博客-CSDN博客_weakhashmap](https://blog.csdn.net/qiuhao9527/article/details/80775524)
  >
  > [一文搞懂WeakHashMap工作原理（java后端面试高薪必备知识点） (baidu.com)](https://baijiahao.baidu.com/s?id=1666368292461068600&wfr=spider&for=pc)
  >
  > [Java篇 - WeakHashMap的弱键回收机制_u014294681的博客-CSDN博客_weakhashmap](https://blog.csdn.net/u014294681/article/details/86522487)

  - 测试weakHashMap

    ```java
    public static void main(String[] args) {
      WeakHashMap<String, Integer> weakHashMap = new WeakHashMap<>();
      for (int i = 0; i < 1024; ++i) {
        String tmp = "String_" + i;
        weakHashMap.put(tmp, i);
        tmp = null; // 注释掉这行，则第一次输出1，后面全部输出2
        System.gc();
        System.out.println(weakHashMap.size());
      }
    }
    ```

    输出最多的是1，偶尔输出0和2

---

- **内存泄漏的第三个常见来源是监听器和其他回调**

  ​	如果实现了一个API，客户端在这个API中注册回调，却没有显式地取消注册，那么除非采取某些动作，否则它们会不断地堆积起来。确保回调立即被当作垃圾回收的最佳方法是只保存它们的弱引用（weak reference），例如，只将它们保存成WeakHashMap中的键。

  > [JAVA回调机制(CallBack)详解 - Bro__超 - 博客园 (cnblogs.com)](https://www.cnblogs.com/heshuchao/p/5376298.html)

## * E8 避免使用终结方法和清除方法

+ 概述

  + **终结方法（finalizer）通常是不可预测的，也是很危险的，一般情况下是不必要的。**
  + **在Java9中用清除方法（cleaner）代替了终结方法。清除方法没有终结方法那么危险，但仍然是不可预测、运行缓慢，一般情况下也是不必要的**。

  + Java的终结方法不能直接想为C++中析构器（destructors）的对应物。

    + C++依赖析构器回收存储空间（内存对象or其他非内存资源如文件描述符等）
    + Java中当一个对戏那个不可达时，垃圾回收器会回收与该对象相关联的存储空间，而其他非内存资源则一般通过try-finally块来完成类似工作（E9）

  + **终结方法和清除方法的缺点在于不能保证被及时执行**。

    一个对象变得不可到达开始，到它的终结方法被执行，所花费的这段时间是任意长的。

    注重时间（time-critical）的任务不应该由终结方法或者清除方法来完成。

  + 执行终结方法和清除方法是垃圾回收算法中的一个主要功能，不同JVM实现中不同。如果程序依赖终结方法、清除方法被执行的时间点，那么不同JVM中运行的表现会截然不同。

  + **Java语言规范不保证终结方法或清除方法的执行（可能永远不执行）**，程序终止时，某些已经无法方法的对象上的终结方法根本没被执行也是完全有可能的。

  + **永远不应该依赖终结方法或清除方法来更新重要的持久状态**。

  + `System.gc`和`System.runFinalization`不保证终结方法或者清除方法被执行（只是增加被执行的机会）。唯一生成保证执行的是`System.runFinalizersOnExit`和`Runtime.runFinalizersOnExit`，但这两个方法有致命缺陷，已经废弃很久了[ThreadStop]。

  + **使用终结方法的另一个问题：如果忽略在终结过程中被抛出的未捕获的异常，该对象的终结过程也会终止**。

    ​	未被捕获的异常会使对象处于破坏的状态（corrupt state），如果另一个线程企图使用这种被破坏的对象，则可能发生不确定的行为。

    + 正常情况下，未被捕获的异常将会使线程终止，并打印出栈轨迹（Stack Trace）
    + **异常发生在终结方法中时，不会打印出栈轨迹，甚至连警告都不会打印**。
    + 清除方法没有该问题，因为使用清除方法的一个类库在控制它的线程。

  + **使用终结方法和清除方法有非常严重的性能损失**。

  + **终结方法存在严重安全问题**。

    终结方法攻击的思想：如果从构造器或者它的序列化对等体（readObject和readResolve方法，详情见12章）抛出异常，恶意子类的终结方法就可以在构造了一部分中途就中途夭折的对象上继续运行。<u>终结方法会将该对象的引用记录在一个静态域中，阻止它被垃圾回收</u>。通过记录到异常的对象，就可以轻松地在这个对象上（终结方法中）调用任何永远不允许在这里出现的方法。

    + **由于终结方法的存在，使得即使构造器抛出异常，也不能保证对象能正常被销毁**。
    + final类不会受到终结方法攻击，因为没有人能够编写出final类的恶意子类。**为了防止非final类受到终结方法攻击，要编写一个空的final的finalize方法**。

  + **让类实现AutoCloseable，来终止对象中封装的资源（文件、线程等）**

    ​	类实现AutoCloseable，并要求客户端在每个实例不再需要使用的时候调用close方法。（一般利用try-with-resources确保终止，即使遇到异常也是如此，E9）

    > 需注意的细节，该实例必须记录自己是否已经被关闭：close方法必须在一个私有域中记录下"该对象已经不再有效"。如果这些方法时在对象已经终止之后才被调用，其他的方法就必须检查这个域，并抛出IllegalStateException异常。

  + 终结方法和清除方法的两个合法用途：

    1. 资源所有者忘记调用它的close方法时，终结方法或清除方法可以充当"安全网"。

       虽然Java规范不保证终结方法或清除方法被执行，但是总比客户端不执行close永远不释放好一些。如果考虑编写这类安全网终结方法，需考虑（性能、安全隐患）代价是否值得。**有些Java类（如FileInputStream、FileOutputStream、ThreadPoolExecutor和java.sql.Conneciton）都具有能充当安全网的终结方法**。

       > [JDK 1.8 API阅读与翻译(2) FileInputStream - 简书 (jianshu.com)](https://www.jianshu.com/p/9586a9d588b1)

    2. **通过终结方法或清除方法回收本地对等体对象（native peer）**。

       ​	本地对等体是一个本地（非Java的）对象（native object），普通对象通过本地方法（native method）委托给一个本地对象。

       ​	因为本地对等体不是一个普通（java）对象，所以垃圾回收器不会知道它，当它的Java对等体被回收的时候，它不会被回收。

       ​	<u>如果本地对等体没有关键资源，并且性能也可以接受的话，那么清除方法或者终结方法正是执行这项任务最合适的工具。如果本地对等体拥有必须被及时终止的资源，或者性能无法接受，那么该类就应该有一个close方法，如前所述</u>。

  + **总而言之，除非是作为安全网，或者是为了终止非关键的本地资源，否则不要使用清除方法，对于在Java9之前的发行版本，则尽量不要使用终结方法。若使用了终结方法或者清除方法，则要注意它的不确定性和性能后果**。

---

+ 举例

  1. 典型错误：使用终结方法或者清除方法来关闭已打开的文件

     ​	打开的文件描述符是很有限的资源，如果系统无法及时运行终结方法或者清除方法，就会导致大量的文件仍然保留在打开的状态，于是当一个程序再也不能打开文件的时候，就可能会运行失败。

     > [linux文件描述符限制和单机最大长连接数_ybxuwei的专栏-CSDN博客_文件描述符上限](https://blog.csdn.net/ybxuwei/article/details/77969032)
     >
     > [网络通信socket连接数上限 - _学而时习之 - 博客园 (cnblogs.com)](https://www.cnblogs.com/sparkleDai/p/7604876.html)

  2. 终结方法队列对象插入速度比回收速度快的少见情况

     ​	**在少数情况下，为类提供终结方法，可能会随意地延迟其实例的回收过程**。书籍作者的同事调试一个长期运行的GUI程序时，程序莫名其妙地出现OutOfMemoryError错误而死掉。分析表明，<u>该应用程序死掉的时候，其终结方法队列中有数千个图形对象正在等待被终结和回收</u>。

     ​	而<u>终结方法线程的优先级比该该应用程序中的其他线程的优先级低得多</u>，所以图形对象的终结速度达不到它们进入队列的速度。

     ​	**Java语言规范不保证哪个线程将会执行终结方法，除了不用终结方法依赖，并没有很轻便的方法能够避免这类问题**。	

     ​	**清除方法稍好一点，类的设计者可以控制自己的清除线程，但清除方法仍然在后台运行，处于垃圾回收器的控制之下，因此不能确保及时清除**。

  3. 错误操作：依赖终结方法或者清除方法来释放共享资源

     由于Java语言规范不保证终结方法的执行，如果依赖终结方法或者清除方法释放诸如数据库上的永久锁等资源，很容易让整个分布式系统跨掉。

     （程序终止时，已经无法访问的对象的终结方法没被执行也是完全可能的）

  4. 终结方法和清除方法的性能损失。

     + 书籍作者通过try-with-resources关闭一个简单的AutoCloseable对象，垃圾回收器将它回收后，这些工作花费时间约12ns；

     + 增加一个终结方法时时间增加到550ns；（终结方法创建和销毁对象比原本慢约50倍，因为终结方法组织了有效的垃圾回收）

     + 采用清除方法清除类的所有实例，比终结方法稍快。如果把清除方法作为安全网（safety net），那能更快一些。这种情况下创建、清除和销毁对象，花费66ns左右。（如果采用清除方法作为安全网，比起慢50倍的终结方法，这个慢5倍）

  5. 清除方法使用例子：

     ​	以简单的Room类为例，假设房间在回收之前必须进行清除。Room类实现了AutoCloseabe；它利用清除方法自动清除安全网的过程只是一个实现细节。与终结方法不同的是，清除方法不会污染类的公有API：

     ```java
     // An autocloseable class using a cleaner as a safety net
     public class Room implements AutoCloseable {
       private static final Cleaner cleaner = Cleaner.create();
       
       // Resource that requires cleaning. Must not refer to Room!
       private static class State implements Runnable {
         int numJunkPiles; // Number of junk piles in this room
         
         State(int numJunkPiles) {
           this.numJunkPiles = numJunkPiles;
         }
         
         // Invoked by close method or cleaner
         @Override public void run() {
           System.out.println("Cleaning room");
           numJunkPiles = 0;
         }
       }
       
       // The state of this room, shared with our cleanable
       private final State state;
       
       // Our cleanable. Cleans the room when it's eligible for gc
       private final Cleaner.Cleanable cleanable;
       
       public Room(int numJunkPiles) {
         state = new State(numJunkPiles);
         cleanable = cleaner.register(this, state);
       }
       
       @Override public void close() {
         cleanable.clean();
       }
     }
     ```

     ​	内嵌的静态类State保存清除方法清除房间所需的资源。在该例中，就是numJunkPiles域，表示房间的杂乱度。更现实地说，它可以是final的long，包含一个指向本地对等体的指针。State实现了Runnable接口，其run方法最多被Cleanable调用一次，后者是我们在Room构造器中用清除器注册State实例时获得的。以下两种情况之一会出发run方法的调用：

     1. 调用Room的close方法（close方法里触发Cleanable的清除方法）
     2. 到了Room实例应该被垃圾回收时，客户端还没调用close方法，清除方法就会（希望如此）调用State的run方法。

     ​	**关键是State实例没有引用它的Room实例**。**如果它引用了，会造成循环，阻止Room实例被垃圾回收（以及防止被自动清除）**。因此<u>State必须是一个静态的嵌套类，因为非静态的嵌套类包含了对其外围实例的引用（E24）</u>。同样，也<u>不建议使用lambda，因为它们很容易捕捉到对外围对象的引用</u>。

  6. Room方法只用作安全网。如果客户端将所有的Room实例化都包在try-with-resource块中，将永远不会请求到自动清除。下面是表现良好的客户端代码示例：

     ```java
     public Class Adult {
       public static void main(String[] args) {
         try(Room myRoom = new Room(7)) {
           System.out.println("Goodbye");
         }
       }
     }
     ```

     正如期望一样，运行程序会打印出“Goodbye”，随后就是“Cleaning room”。但下面这个程序就不保证会输出“Cleaning room”

     ```java
     public class Teenager{
       public static void main(String[] args) {
         new Room(99);
         System.out.println("Peace out");
       }
     }
     ```

     ​	上诉代码在书籍作者机器上没打印出“Cleaning room”就退出程序了。这就是前面提及的（终结方法、清除方法的）不可预见性。

     ​	**Cleaner规范指出："清除方法在System.exit期间的行为是与实现相关的。不确保清除动作是否会被调用。"虽然规范没有指明，但其实对于正常的程序退出也是如此**。在作者的机器上，只要在main方法加上`System.gc()`就能在退出之前打印"Cleaning room"。（但不保证所有人操作时都如此）

## E9 try-with-resources 优先于 try-finally

p37
