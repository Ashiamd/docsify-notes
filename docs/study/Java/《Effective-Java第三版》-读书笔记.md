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

p24
