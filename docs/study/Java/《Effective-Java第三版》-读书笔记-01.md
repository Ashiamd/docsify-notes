# 《Effective-Java第三版》-读书笔记-01

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

+ 概述

  ​	Java类库中包括许多必须通过调用close方法来手工关闭的资源。例如InputStream、OutputStream和java.sql.Connection。

  ​	**虽然其中的许多资源都是用终结方法作为安全网，但是效果并不理想（E8）**。

  ​	根据经验，try-finally语句是确保资源会被适时关闭的最佳方法，就算发生异常或者返回也一样。

  + 在需要关闭的资源较多，嵌套使用try-finally可读性差，还需要考虑调用在finally调用close方法时抛出异常的情况。

  + Java7引入try-with-resources语句，解决try-finally问题。要使用这个构造的资源，必须要实现AutoCloseable接口，其中包含了单个返回void的close方法。

    Java类库和第三方类库中许多类和接口，都实现或扩展了AutoCloseable接口。**如果编写一个类，它代表的是必须被关闭的资源，那么这个类也应该实现AutoCloseable**。

  + **在处理必须关闭的资源时，始终优先考虑用try-with-resources，而不是用try-finally**。
    + 简洁、清晰
    + 产生的异常更有价值

  ---

  + 举例：

    文件读取一行，然后释放资源。分别用try-finally和try-with-resources实现。

    1. try-finally

       ```java
       // try-finally - No longer the best way to close resources!
       static String firstLineOfFile(String path) throws IOException {
         BufferedReader br = new BufferedReader(new FileReader(path));
         try{
           return br.readLine();
         } finally {
           br.close();
         }
       }
       ```

    2. try-with-resources

       ```java
       // try-with-resources - the best way to close resources!
       static String firstLineOfFile(String path) throws IOException {
         try (BufferedReader br = new BufferedReader(new FileReader(path))) {
           return br.readLine();
         }
       }
       ```

    ​	上述try-finally实现中，如果底层的物理设备异常，那么调用readLine就会抛出异常，同理调用close也会抛出异常。<u>在这种情况下，第二个异常完全抹除了第一个异常</u>。在异常堆栈轨迹中，完全没有关于第一个异常的记录，这在现实的系统中会导致调试变得非常复杂，因为通常需要看到第一个异常才能诊断出问题何在。虽然可以通过编程来禁止第二个异常，保留第一个异常，但事实上没有人会这么做，因为实现繁琐。

    ​	而使用try-with-resources，假设同样readLine和（不可见的）close都抛出异常，后一个异常就会被禁止，以保留第一个异常。

    > 事实上，为了保留你想要看到的那个异常，即便多个异常都可以被禁止。这些被禁止的异常并不是简单地被抛弃了，而是会被打印在堆栈轨迹中，并注明它们是被禁止的异常。通过编程调用getSuppressed方法还可以访问到它们，getSuppressed方法也已经添加在Java7的Trowabe中了。

# 3、对于所有对象都通用的方法

​	尽管Object是一个具体类，但设计它主要是为了扩展。它所有的非final方法（equals、hashCode、toString、clone和fianlize）都有明确的通用约定（general contract），因为它们设计成是要被覆盖（override）的。

​	E8讨论的finalize本章节不再讨论；Comparable。compareTo虽然不是Object方法，但是本章也对它进行讨论，因为它也具有类似的特征。

## * E10 覆盖equals时请遵守通用约定

+ 概述

  ​	覆盖equals方法容易犯错，避免的方式即不覆盖equals方法，在这种情况下，类的每个实例都只与它自身相等。

  ​	如果满足了以下**任何一个条件**，这就正是所期望的结果：

  + **类的每个实例本质上都是唯一的**。

    ​	对于代表活动实体而不是值（value）的类来说确实如此，例如Thread。Object提供的equals实现对于这些类来说正是正确的行为。

  + **类没有必要提供"逻辑相等"（logical equality）的测试功能**。

    ​	例如，Java.util.regex.Pattern可以覆盖equals，以检查两个Pattern实例是否代表同一个正则表达式，但是设计者并不认为客户需要或者期望这样的功能。在这种情况下，从Object继承得到的equals实现已经足够了。

  + **超类已经覆盖了equals，超类的行为对于这个类也是合适的**。

    ​	例如，大多数的Set实现都从AbstractSet继承equals实现，List实现从AbstractList继承equals实现，Map实现从AbstractMap继承equals实现。

  + **类是私有的，或者是包级私有的，可以确定它的equals方法永远不会被调用**。

    ​	如果你非常想要规避风险，可以覆盖equals方法，以确保它不会被意外调用：

    ```java
    @Override public boolean equals(Object o) {
      throw new AssertionError(); // Method is never called
    }
    ```

---

- 需要覆盖equals方法的场景

  ​	类具有"逻辑相等（logical equality）"概念，且超类没有覆盖equal。（这通常属于"值类value class"的情形）。

  ​	有一种"值类"不需要覆盖equals方法，即用实例受控（E1）确保"每个值最多只存在一个对象"的类。枚举类型（E34）就属于这种类。对于这种类而言，逻辑相等 = 对象等同，因此Object的equals方法等同于逻辑意义上的equals方法

  ```java
  // Object.java
  public boolean equals(Object obj) { return (this == obj); }
  ```

  > 值类仅仅是一个表示值的类，比如Integer或者String

---

* 覆盖equals方法的通用约定

  ​	下面是约定的内容，来自Object的规范。

  ​	equals方法实现了**等价关系（equivalence relation）**，其属性如下：

  + **自反性（reflexive）**：对于任何非null的引用值x，`x.equals(x)`必须返回true。
  + **对称性（symmetric）**：对于任何非null的引用值x和y，当且仅当`y.equals(x)`返回true时，`x.equals(y)`必须返回true。
  + **传递性（transitive）**：对于任何非null的引用值x、y和z，如果`x.equals(y)`返回true，且`y.equals(z)`也返回true，那么`x.equals(z)`也必须返回true。
  + **一致性（consistent）**：对于任何非null的引用值x和y，只要equals的比较操作在对象中所用的信息没有被修改，多次调用x.equals(y)就会一致地返回true，或者一致地返回false。

  + **非空性（Non-nullity）**：对于任何非null的引用值x，`x.equals(null)`必须返回false。

---

+ 等价关系是什么

  ​	不严格地说，它就是一个操作符，将一组元素划分到其元素与另一个元素等价的分组中。这些分组被称为等价类（equivalence class）。从用户的角度来看，对于有用的equals方法，每个等价类中的所有元素都必须是可交换的。

----

+ **equals的5个属性分析**：

  1. 自反性（reflexive）：一般不会违背这条。假如违背了，那么添加该类的实例到集合中，该集合的contains方法会表示不包含刚才添加的实例。

  2. 对称性（symmetric）：不注意的话就可能违反。

     ```java
     // Broken - violates symmetry!
     public final class CaseInsensitiveString {
       private final String s;
       
       public CaseInsensitiveString(String s) {
         this.s = Objects.requireNonNull(s);
       }
       
       // Broken - violates symmetry!
       @Override public boolean equals(Object o) {
         if(o instanceof CaseInsensitiveString)
           return s.equalsIngnoreCase(((CaseInsensitiveString) o).s);
         if(o instanceof String) // One-way interoperability!
           return s.equalsIgnoreCase((String) o);
         return false;
       }
       ... // Remainder omitted
     }
     ```

     ​	比如某个类重写了equals方法，用于判断不区分大小写的字符串是否相等。那么该类实例调用equals方法时，传入原字符串不相等但是不区分大小写时相等的String实例，返回true；反过来调用String实例的equals方法则返回false。

     ​	此时如果把该类对象放入集合，调用集合的contains方法（传入不区分大小写时才能相等的String实例），不能保证返回true或者false（不同JDK实现或许不同，但是作者的jdk版本返回false）

     ​	<u>一旦违反了equals约定，当其他对象面对你的对象时，你完全不知道这些对象的行为会怎么样</u>。

     ​	为解决问题，该写equals方法如下，该类不再用于和String进行比较

     ```java
     @Override public boolean equals(Object o) {
       return o instanceof CaseInsensitiveString && 
         ((CaseInsensitiveString) o).s.equalsIgnoreCase(s);
     }
     ```

  3. 传递性（transitivity）：这个也是容易违反的。**子类增加的信息会影响equals的比较结果**。

     ```java
     public class Point {
       private final int x;
       private final int y;
       
       public Point(int x, int y) {
         this.x = x;
         this.y = y;
       }
       
       @Override public boolean equals(Object o) {
         if(!(o instanceof Point))
           return false;
         Point p = (Point)o;
         return p.x == x && p.y == y;
       }
       
       ... // Remainder omitted
     }
     ```

     子类ColorPoint扩展Point，并编写equals如下

     ```java
     public class ColorPoint extends Point {
       private final Color color;
       
       public ColorPoint(int x, int y, Color color) {
         super(x, y);
         this.color = color;
       }
       
       // Broken - violates symmetry!
       @override public boolean equals(Object o) {
         if(!(o instanceof ColorPoint))
           return false;
         return super.equals(o) && ((ColorPoint) o).color == color;
       }
     }
     ```

     ​	上诉写法，在比较Point和ColorPoint的时候会得到不正确的结果。

     修改如下时（依旧错误）

     ```java
     @Override public boolean equals(Object o) {
       if(!(o instanceof Point))
         return false;
       // If o is a normal Point, do a color-blind comparison
       if(!(o instanceof ColorPoint))
         return o.equals(this);
       
       // o is a ColorPoint; do a full comparison
       return super.equals(po) && ((ColorPoint) o).color == color;
     }
     ```

     上诉实现，提供了对称性，但是牺牲了传递性

     ```java
     ColorPoint p1 = new ColorPoint(1, 2, Color.RED);
     Point p2 = new Point(1, 2);
     ColorPoint p3 = new ColorPoint(1, 2, Color.BLUE);
     ```

     此时`p1.equals(p2)`和`p2.equals(p3)`返回true，但是`p1.equals(p3)`返回false，违反了传递性。

     <u>此外，这种实现方式可能导致无限递归问题</u>。例如有Point有两个子类都以上诉方式实现equals方法，那么调用equals方法比较这两个子类，将抛出StackOverflowError异常。

     ​	这是面向对象语言中关于等价关系的一个基本问题。**我们无法在扩展可实例化的类的同时，既增加新的值组件，同时又保留equals约定**，除非愿意放弃面向对象的抽象所带来的优势。

     ​	虽然没有一种令人满意的方法可以既扩展不可实例化的类，又增加值组件，但是有不错的权宜之计：遵从（E18）"**复合优先于继承**"的建议。

     ​	不再让ColorPoint扩展Point，而是在ColorPoint中加入一个私有的Point域，以及一个公有的视图（view）方法（E6），此方法返回一个与该有色点处在相同位置的普通Poitn对象：

     ```java
     // Adds a value component without violating the equals contract
     public class ColorPoint {
       private final Point point;
       private final Color color;
       
       public ColorPoint(int x, int y, Color color) {
         point = new Point(x,y);
         this.color = Objects.requireNonNull(color);
       }
       
       /**
        * Returns the point-view of this color point.
        */
       public Point asPoint(){
         return point;
       }
       
       @Override public boolean equals(Object o) {
         if(!(o instanceof ColorPoint))
           return false;
         ColorPoint cp = (ColorPoint) o;
         return cp.point.equals(point) && cp.color.equals(color);
       }
       
       ... // Remainder omitted
     }
     ```

     > Java平台类库中，有一些类扩展了可实例化的类，并添加了新的值组件。例如，java.sql.Timestamp对java.util.Date进行了扩展，并增加了nanoseconds域。**Timestamp的equals实现确实违反了对称性**，如果Timestamp和Date对象用于同一个集合中，或者以其他方式被混合在一起，则会引起不正确的行为。Timestamp类的这种行为时各错误，不值得效仿。

     > 注意，**你可以在一个抽象（abstract）类的子类中增加新的值组件且不违反equals约定**。对于（E23）的建议而得到的那种类层次结构来说，这一点非常重要。
     >
     > 例如，你可能有一个抽象的Shape类，它没有任何值组件，Circle子类添加了一个radius域，Rectangle子类添加了length和width域名。**只要不可能直接创建超类的实例，前面的所述种种问题就不会发生。**

  4. 一致性（Consistency）：可变对象在不同的时候与不同的对象相等，而不可变对象则不会这样。

     在编写类的时候，应该仔细考虑其是否不可变（E17）。<u>如果是不可变类，需保证equals满足这样的限制条件：相等的对象永远相等，不相等的对象永远不相等。</u>

     ​	**无论类是否不可变，都不要使equals方法依赖于不可靠的资源**。

     > java.net.URL的equals方法，依赖于对URL主机IP地址的比较。将一个主机名转换成IP地址可能需要访问网络，随着时间的推移，就不能确保会产生相同的结果，即IP地址可能发生了改变。这样会导致URL equals方法违反euqals约定，在实践中可能引发一些问题。
     >
     > URL euqals方法的行为是一个大错误且不应该被模范。
     >
     > 为了避免这类问题，equals方法应该对驻留在内存中的对象执行确定性的计算。

  5. 非空性（Non-nullity）：所有对象都不能等于null。

     很多类的equals方法都通过一个显式的null来防止`o.equals(null)`意外返回true或者抛出NullPointerException。

     ```java
     @Override public boolean equals(Object o) {
       if(o == null)
         return false;
       ...
     }
     ```

     ​	这项测试是不必要的。<u>为了测试参数的等同性，equals方法必须先把参数转化成适当的类型，以便可以调用它的访问方法，或者访问它的域。在进行转化之前，equals方法必须使用instanceof操作符，检查其参数的类型是否正确</u>：

     ```java
     @Override public boolean equals(Object o) {
       if(!(o instanceof MyType))
         return false;
       MyType mt = (MyType) o;
       ...
     }
     ```

     ​	<u>如果漏掉了这一步的类型检查，并且传递给equals方法的参数又是错误的类型，那么equals方法将会抛出CLassCastException异常，这就违反了equals约定。但是，如果instanceof的第一个操作数为null，那么不管第二个操作数是哪种类型，instanceof操作符都指定应该返回false。因此，如果把null传给equals方法，类型检测就会返回false，所以不需要显式的null检查</u>。

---

- **如何实现高质量equals方法**

  1. **使用\=\=操作符检查"参数是否为这个对象的引用"**。

     如果是，返回true。这只不过是一种性能优化，如果比较操作有可能比较昂贵，就值得这么做。

  2. **使用instanceof操作符检测"参数是否为正确的类型"**。

     如果不是，返回false。<u>一般来说，所谓"正确的类型"是指equals方法所在的那个类。某些情况下，是指该类所实现的某个接口。如果类实现的接口改进了equals约定，允许在实现了该接口的类之间比较，那么就使用接口。集合接口如Set、List、Map.Entry具有这样的特性</u>。

  3. **把参数转换成正确的类型**。

     因为转换之前进行过instanceof测试，所以确保会成功。

  4. **对于该类中的每个"关键"（signficant）域，检查参数中过的域是否与该对象中对应的域相匹配**。

     如果这些测试全部成功，则返回true；否则返回false。

     + 如果第2步中的类型是个接口，就必须通过接口方法访问参数中的域；如果该类型是个类，也许就能够直接访问参数中的域，取决于它们的可访问性。

     + 对于非浮点数类型（既不是float也不是double类型）的基本类型域，可以使用==操作符进行比较

     + 对于对象引用域，可以递归地调用equals方法

     + 对于float域，可以使用静态`Float.compare(float, float)`方法

     + 对于double域，可以使用静态`Double.compare(double, double)`方法

       > 对于float和double域进行特殊处理是必要的，因为存在着Float.NaN，-0.0f以及类似的double常量；详细信息参考JLS 15.21.1 或者 Float.equals的文档。
       >
       > 使用Float.compare或者Double.compare每次比较都会自动装箱，会导致性能下降。

     + 对于数组域，需要把以上指导原则应用到每一个元素中。如果数组域中每个元素都很重要，就可以使用其中一个Arrays.equals方法。

     > <u>有些对象引用域包含null可能是合法的，为了避免可能导致NullPointerException异常，则使用静态方法`Objects.equals(Object, Object)`来检查这类域的等同性</u>。

     ​	域的比较顺序可能会影响equals方法的性能。为了获得最佳的性能，应该**最先比较最有可能不一致的域，或者是开销最低的域**，最理想的情况是两个条件同时满足的域。

     ​	**不应该比较那些不属于对象逻辑状态的域**，例如用于同步操作的Lock域。

     ​	<u>也不需要比较衍生域（derived field），因为这些域可以由"关键域"（significant field）计算获得，但是这样做有可能提高equals方法的性能。如果衍生于代表了整个对象的描述信息，比较这个域可以节省在比较失败时取比较实际数据所需要的开销。例如，假设有一个Polygon类，并缓存了该面积。如果两个多边形有着不同的面积，就没有必要再去比较它们的边和定点</u>。

     ​	**在编写完equals方法之后，需要确认：是否对称的、传递的、一致的**（当然equals也需要满足其他两个特性：自反性、非空性，这两个特性通常会自动满足），并且需要编写单元测试验证这些特性，除非使用AutoValue生成equals方法，这种情况下就可以放心地省略测试。

---

+ 告诫

  1. **覆盖equals时总要覆盖hashCode**（E11）。
  2. **不要企图让equals方法过于智能**。
  3. **不要将equals声明中的Object对象替换成其他类型**。（替换了，则没有覆盖Object.equals）

  编写和测试equals（和hashCode）都是繁琐的。可以利用Google开源的AutoValue框架，它会自动生成这些方法。大多数情况下，AutoValue生成的方法本质上和亲自编写的方法是一样的。

  ​	总而言之，不要轻易覆盖equals方法，除非迫不得已。如果覆盖equals方法，需要比较这个类的所有关键域，并且查看它们是否遵守equals合约的所有五个条款。

## E11 覆盖equals时总要覆盖hashCode

- 概述

  **在每个覆盖了equals方法的类中，都必须覆盖hashCode方法**。

  ​	如果不这样做就违反了hashCode的通用约定，从而导致该类无法结合所有基于散列的集合一起正常运作，这类集合包括HashMap和HashSet。下面是约定的内容，摘自Object规范：

  + <u>在应用程序的执行期间，只要对象的equals方法的比较操作用到的信息没有被修改，那么对同一个对象的多次调用，hashCode方法都必须始终返回返回同一个值</u>。在一个应用程序与另一个程序的执行过程中，执行hashCode方法所返回的值可以不一致。
  + <u>如果两个对象根据equals（Object）方法比较是相同的，那么调用这两个对象中的hashCode方法都必须产生同样的整数结果</u>。
  + <u>如果两个对象根据equals（Object）方法比较是不相等的，那么调用这两个对象中的hashCode方法，则不一定要求hashCode方法必须产生不同的结果</u>。但是程序员应该知道，给不相等的对象产生截然不同的整数结果，有可能提高散列表（hash table）的性能。

  **因没有覆盖hashCode而违反的关键约定是第二条：相等的对象必须具有相等的散列码（hash code）**。

  ​	根据类的equals方法，两个截然不同的实例在逻辑上有可能是相等的，但是根据Object类的hashCode方法，它们仅仅是两个没有任何共同之处的对象。因此对象的hashCode方法返回两个看起来是随机的整数，而不是根据第二个约定所要求的那样，返回两个相等的整数。

---

- 合理实现hashCode方法

  ​	理想情况下，散列函数应该把集合中不相等的实例均匀分布到所有可能的int值上。想要完全达到这种理想的情况是非常困难的。相对接近理想情况的一种简单解决方法如下：

  ```none
  1. 声明一个int变量并命名为result，将它初始化为对象中第一个关键域的散列码c，如步骤2.a中计算所示（如第10条所述，关键域是指影响equals比较的域）。
  
  2. 对象中剩下的每一个关键域f都完成以下步骤：
  	a. 为该域计算int类型的散列码c：
  		1. 如果该域是基本类型，则计算Type.hashCode(f)，这里的Type是装箱基本类型的类，与f的类型相对应。
  		2. 如果该域是一个对象引用，并且该类的equals方法通过递归地调用equals的方法来比较这个域，则同样为这个域递归地调用hashCode。如果需要更复杂的比较，则为这个域计算一个"范式"（canonical representation），然后针对这个范式调用hashCode。如果这个域的值为null，则返回0（或者其他某个常数，但通常是0）。
  		3. 如果该域是一个数组，则要把每一个元素当作单独的域来处理。也就是说，递归地应用上述规则，对每个重要的元素计算一个散列码，然后根据步骤2.b中的做法把这些散列值组合起来。如果数组域中没有重要的元素，可以使用一个常量，但最好不要用0。如果数组域中的所有元素都很重要，可以使用Arrays.hashCode方法。
  	b. 按照下面的公式，把步骤2.a中计算得到的散列码c合并到result中:
  		result = 31 * result + c;
  		
  3. 返回result。
  ```

  > 在散列码的计算过程中，可以把衍生域（derived field）排除在外。换句话说，如果一个域的值可以根据参与计算的其他域值计算出来，则可以把这样的域排除在外。必须排除equals比较计算中没有用到的任何域，否则很有可能违反hashCode约定的第二条。

  > 步骤2.b中的乘法部分使得散列值依赖于域的顺序，如果一个类包含多个相似的域，这样的乘法运算就会产生一个更好的散列函数。例如，如果String散列函数省略了这个乘法部分，那么只是字母顺序不同的所有字符串将会有相同的散列码。
  >
  > 之所以选择31，因为它是一个奇素数。如果乘数是偶数，并且乘法溢出的话，信息就会丢失，因为与2相乘等价于移位运算。使用素数的好处并不是很明显，但是习惯上都使用素数来计算散列结果。<u>31有个很好的特性，即用移位和减法来替代乘法</u>，可以得到更好的性能：`31 * i == ( i << 5 ) - i`。现代的虚拟机可以自动完成这种优化。

  ​	<u>Objects类有一个静态方法，它带有任意数量的对象，并为它们返回一个散列码</u>。这个方法名为hash，是让你只需要编写一行代码的hashCode方法，与前面描述的实现方法质量相当。缺点是运行速度更慢一些，因为会引发数组的创建，以便传入数量可变的参数，如果参数中有基本类型，还需要装箱和拆箱。建议只将这类散列函数用于不太注重性能的情况。

  ​	**如果类不可变，并且计算散列码的开销比较大，可以考虑把散列码换存在对象内部，而不是每次请求的时候都重新计算散列码**。如果你觉得这种类型的大多数对象会被用作散列键（hash keys），就应该在创建实例的时候计算散列码。否则，可以选择**"延迟初始化"（lazy initialize）散列码**，即一直到hashCode被第一次调用的时候才初始化（E83）。

  ​	**不要试图从散列码计算中排除掉一个对象的关键域来提高性能**。虽然这样的到的散列函数运行起来可能更快，但是可能导致散列表慢到根本用不了。在选择忽略的区域之中，有些实例要是区别很大，那么散列函数就会把所有这些实例映射到极少数的散列码上，原本线性级时间运行的程序，将编程平方级时间运行。

  > 这不只是理论问题。在Java2发行版之前，一个String散列函数最多只能使用16个字符，若长度少于16个字符就计算所有的字符，否则就从第一个字符开始，在整个字符串中间隔均匀地选取样本进行计算。对于像URL这种层次状名称的大型集合，该散列函数正好表现出了这里所提到的病态行为。

  ​	**不要对hashCode方法的返回值做出具体的规定，因此客户端无法理所当然地依赖它；这样可以为修改提供灵活性**。

  > Java类库中很多类，比如String和Integer，都可以把它们的hashCode方法返回的确切值规定为该实例值的一个函数。一般来说，这并不是个好主意，这样严格限制了未来版本中改进散列函数的能力。如果没有规定散列函数的细节，那么当你发现了它的内部缺陷时，或者发现了更好的散列函数时，就可以在后面的发行版本中修正它。

---

+ hashCode实现样例

  + 前面说的用到奇素数的实现方式：

    ```java
    // Typical hashCode method
    @Override public int hashCode() {
      int result = Short.hashCode(areaCode);
      result = 31 * result + Short.hashCode(prefix);
      result = 31 * reuslt + Short.hashCode(lineNum);
      return result;
    }
    ```

    ​	前面描述的hashCode实现方法能获得相当好的散列函数，质量堪比Java平台类库的值类型中提供的散列函数，对于绝大多数应用程序而言已经足够了。

  > 如果执意要让散列函数尽可能不冲突，可参考`Guava's com.google.common.hash.Hashing`

  + 使用Objects类的hash方法实现散列函数的例子如下：

    ```java
    // One-line hashCode method - mediocre performance
    @Override public int hashCode() {
      return Objects.hash(lineNum, prefix, areaCode);
    }
    ```

  + "延迟初始化"（lazy initialize）散列码，直到hashCode第一次被调用才初始化——杨例：

    ```java
    // hashCode method with lazily initialized cached hash code
    private int hashCode; // Automatically intialized to 0
    
    @Override public int hashCode() {
    	int result = hashCode;
      if(reuslt == 0) {
        result = Short.hashCode(areaCode);
        reuslt = 31 * reuslt + Short.hashCode(prefix);
        result = 31 * result + Short.hashCode(lineNum);
        hashCode = reuslt;
      }
      return result;
    }
    ```

    **注意：hashCode域的初始值（在本例中是0）一般不能称为创建的实例的散列码**

---

+ 小结：

  覆盖equals方法时必须覆盖hashCode，否则程序无法正常运行。

  hashCode方法必须遵守Object规定的通用约定。

## E12 始终要覆盖toString

+ 概述

  Object提供的默认toString实现，返回`类的名称@散列码`。

  + 当对象被传递给println、printf、字符串联操作符（+）以及assert，或者被调试器打印出来时，toString方法会被自动调用。
  + toString方法应该返回对象中包含的所有值得关注的信息
  + 在静态工具类（E4）中编写toString方法是没有意义的。也不要在大多数枚举类型（E34）中编写toString方法，因为Java已经提供了非常完美的方法。
  + 一般而言需要在编写的每一个可实例化的类中覆盖Object的toString实现，除非已经在超类中这么做了。

## * E13 谨慎地覆盖clone

+ 概述

​	Cloneable接口的目的是作为对象的一个mixin接口（mixin interface）（E20），表明这样的对象允许克隆（clone）。

​	<u>遗憾的是，它并没有成功达到这个目的。它的主要缺陷在于少了一个clone方法，而Object的clone方法是受保护的（protected）。如果不借助反射（reflection）（E65），就不能仅仅因为一个对象实现了Cloneable，就调用clone方法</u>。**即使是反射调用也可能会失败，因为不能保证对象一定具有可访问的clone方法**。

​	尽管存在这样或那样的缺陷，这项设施仍然被广泛使用，因此值得我们进一步了解。下面将介绍如何实现一个行为良好的clone方法，并讨论何时适合这么做，同时简单介绍其他可替代做法。

> 既然Cloneable接口并没有包含任何方法，那有什么作用？它决定了Object中受保护的clone方法实现的行为：**如果一个类实现了Cloneable，Object的clone方法就返回该对象的逐域拷贝，否则就会抛出`CloneNotSupportedException`异常**。
>
> 这是接口的一种极端非典型的用法，也不值得效仿。
>
> 通常情况下，实现接口是为了表明类可以为它的客户做些什么，然而，对于Cloneable接口，它改变了超类中受保护的方法的行为。

​	虽然规范中没有明确指出，**事实上，实现Cloneable接口的类是为了提供一个功能适当的公有的clone方法**。

​	为了达到这个目的，类及所有超类都必须遵守一个相当复杂的、不可实施的，并且基本上没有文档说明的协议。由此得到一种语言之外的（extra linguistic）机制：**它无须调用构造器就可以创建对象**。

---

+ clone方法的通用约定

​	clone方法的通用约定是非常弱的，下面是来自Object规范中的约定内容：

​	创建和返回一个该对象的一个拷贝。这个"拷贝"的精确含义取决于该对象的类。

​	**一般的含义是，任何对于对象x，表达式`x.clone() != x`将会返回结果true，并且表达式`x.clone().getClass() == x.getClass()`将会返回结果true，但这些都不是绝对的要求**。

​	**虽然通常情况下，表达式`x.clone().equals(x)`将会返回结果true，但是，这也不是一个绝对的要求**。

​	**按照约定，这个方法返回的对象应该通过调用`super.clone`获得。如果类及其超类（Object除外）遵守这一约定，那么：`x.clone().getClass() == x.getClass()`** 

​	<u>按照约定，返回的对象应该不依赖于被克隆的对象。为了成功地实现这种独立性，可能需要在`super.clone`返回对象之前，修改对象的一个或更多个域</u>。

> ​	这种机制大体上类似于自动的构造器调用链，只不过它不是强制要求的：如果类的c1one方法返回的实例不是通过调用`super.clone`方法获得，而是通过调用构造器获得，编译器就不会发出警告，但是该类的子类调用了`super.clone`方法，得到的对象就会拥有错误的类，并阻止了clone方法的子类正常工作。
>
> ​	如果 final类覆盖了clone方法，那么这个约定可以被安全地忽略，因为没有子类需要担心它。如果 final类的clone方法没有调用`super.clone`方法，这个类就没有理由去实现`Cloneable`接口了，因为它不依赖于Object克隆实现的行为。

+ 如果你希望在一个类中实现Cloneable接口，并且它的超类都提供了行为良好的clone方法啊。首先，调用`super.clone`方法。因此得到的对象将是原始对象功能完整的克隆（clone）。在这个类中声明的域将等同于被克隆对象中相应的域。如果每个域包含一个基本类型的值，或者包含一个指向不可变对象的引用，那么被返回的对象则可能正是你所需要的对象，在这这种情况下不需要再做进一步处理。

+ **不可变的类永远都不应该提供clone方法**，因为它只会激发不必要的克隆。

+ 如果对象中包含的域引用了可变的对象，则clone方法的实现不能仅是调用`super.clone`。

  > 引用类型只复制了引用，被引用的对象改变时，clone出来的对象也受影响。（即浅拷贝，而非深拷贝）

+ **实际上，clone方法就是另一个构造器；必须确保它不会伤害到原始的对象，并确保正确地创建被克隆对象中的约束条件（invariant）**。

+ clone方法禁止给final域赋新值。就像序列化一样，**Cloneable架构与引用可变对象的final域的正常用法是不相兼容的**。

+ 像构造器一样，clone方法也不应该在构造的过程中，调用可以覆盖的方法（E19）。如果clone调用了一个在子类中被覆盖的方法，那么在该方法所在的子类有机会修正它在克隆对象中的状态之前，该方法就会先被执行，这样很有可能会导致克隆对象和原始对象之间的不一致。

+ Object的clone方法被声明为可抛出`CloneNotSupportedException`异常，但是覆盖版本的clone方法可以忽略这个声明。**公有的clone方法应该省略throws声明**，因为不会抛出受检异常的方法使用起来更加轻松（E71）。

+ <u>为继承（E19）设计类有两种选择，但是无论选择其中的哪一种方法，这个类都不应该实现Cloneable接口</u>。你可以选择模拟Object的行为：实现一个功能适当的受保护的clone方法，它用该被声明抛出`CloneNotSupportedException`异常。这样可以使子类具有实现或不实现Cloneable接口的自由，就仿佛它们直接扩展了Object一样。或者，也可以选择不去实现一个有效的clone方法，并防止子类去实现它，只需要提供下列退化了的clone实现即可：

  ```java
  // clone method for extendable class not supporting Cloneable
  @Override
  protected final Object clone() throws CloneNotSupportedException {
    throw new CloneNotSupportedException();
  }
  ```

+ **编写线程安全的类如果准备实现`Cloneable`接口，需要记住clone方法必须得到严格的同步，就像其他方法一样（E78）。Object的clone方法没有同步**，需要自己提供实现。

+ **在数组上调用clone返回的数组，其编译时的类型与被克隆数组的类型相同。这是复制数组的最佳习惯做法。事实上，数组是clone方法唯一吸引人的用法。**

---

+ clone实现-举例

  1. （E11）中的PhoneNumber类实现Cloneable接口。

     ```java
     @override public PhoneNumber clone() {
       try {
         return (PhoneNumber) super.clone();
       } catch (CloneNotSupportedException e) {
     		throw new AssertionError(); // Can't happen 
       }
     }
     ```

     ​	为了让这个方法生效，应该修改PhoneNumber的类声明实现为实现Cloneable接口。虽然Object的clone方法返回的是Object，但这个clone方法返回的却是PhoneNumber。这么做是合法的，也是我们所期待的，因为**Java支持协变返回类型（covariant return type）**。换句话说，目前覆盖方法的返回值类型可以是被覆盖方法的返回类型的子类了。这样在客户端中就不必进行转换了。我们必须在返回结果之前，先将super.clone从Object转换成PhoneNumber，当然这种转化是一定会成功的。

     > [Java之协变返回类型理解和简单实例_阿毅-CSDN博客_协变返回类型](https://blog.csdn.net/huangwenyi1010/article/details/53454542)
     >
     > [java - 什么是协变返回类型？ - ITranslater](https://www.itranslater.com/qa/details/2325839725233439744)
     >
     > **简言之，子类/实现类重写方法时，允许返回值为超类的子类**。

  2. （E7）中Stack类的clone，为了使Stack类中的clone方法正常工作，必须要拷贝栈的内部信息。最容易的做法是，在elements数组中递归地调用clone：

     ```java
     // Clone method for class with references to mutable state
     @Override public Stack clone() {
       try {
         Stack result = (Stack) super.clone();
         result.elements = elements.clone();
         return result;
       } catch (CloneNotSupportException e) {
         throw new AssertionError();
       }
     }
     ```

     ​	**注意，我们不一定要将`elements.clone()`的结果转换成`Object[]`。在数组上调用clone返回的数组，其编译时的类型与被克隆数组的类型相同。这是复制数组的最佳习惯做法。事实上，数组是clone方法唯一吸引人的用法**。

     > 还要注意如果elements域是final的，上述方案就不能正常工作，因为clone方法是被禁止给final域赋新值的。这是个根本的问题：就像序列化一样，**Cloneable架构与引用可变对象的final域的正常用法是不相兼容的**，除非在原始对象和克隆对象之间可以安全地共享此可变对象。为了使类成为可克隆的，可能有必要从某些域中去掉final修饰符。

---

+ 替代clone的方案

  ​	如果扩展一个实现了Cloneable接口的类，那么除了实现一个良好的clone方法外，没有其他选择。

  **对象拷贝的更好方法是提供一个拷贝构造器（copy constructor）或拷贝工厂（copy factory）**。

  拷贝构造器只是一个构造器，它唯一的参数类型就是包含该构造器的类，例如：

  ```java
  // Copy constructor
  public Yum(Yum yum) {...};
  ```

  拷贝工厂是类似于拷贝构造器的静态工厂（E1）

  ```java
  public static Yum newInstance(Yum yum) {...};
  ```

  拷贝构造器的做法，及其静态工厂方法的变形，都比Cloneable/clone方法具有更多的优势：

  + 不依赖于某一种很有风险的、语言之外的对象创建机制；
  + 不要求遵守尚未制定好文档的规范；
  + 不会与final域的正常使用发生冲突；
  + 不会抛出不必要的受检异常；
  + 不需要进行类型转化。

  > 甚至，拷贝构造器或者拷贝工厂可以带一个参数，参数类型是该类所实现的接口。例如，**按照惯例所有通用集合实现都提供了一个拷贝构造器，其参数类型为Collection或者Map接口**。
  >
  > <u>基于接口的拷贝构造器和拷贝工厂（更准确的叫法应该是转换构造器（conversion constructor）和转换工厂（conversion  factory）），允许客户选择拷贝的实现类型，而不是强迫客户接受原始的实现类型</u>。
  >
  > 例如，假设你有一个Hashset:s，并且希望把它拷贝成一个TreeSet。clone方法无法提供这样的功能，但是用转换构造器很容易实现：newTreeSet<>(s)

---

+ 小结

  ​	**既然所有的问题都与Cloneable接口有关，新的接口就不应该扩展这个接口，新的可扩展的类也不应该实现这个接口**。

  ​	虽然final类实现 Cloneable接口没有太大的危害，这个应该被视同性能优化，留到少数必要的情况下才使用（E67）。

  ​	**总之，复制功能最好由构造器或者工厂提供。这条规则最绝对的例外是数组，最好利用 clone方法复制数组。**

  *（如前文所言，**在数组上调用clone返回的数组，其编译时的类型与被克隆数组的类型相同**。这是复制数组的最佳习惯做法。事实上，数组是clone方法唯一吸引人的用法）*

## * E14 考虑实现Comparable接口

+ 概述

​	compareTo方法并没有在Object类中声明，它是Comparable接口中唯一的方法。

​	compareTo方法，不但允许进行简单的等同性比较，而且允许执行顺序比较，而且与Object的equals方法具有相似的特征，是个泛型（generic）。类实现了Comparable接口，就表明它的实例具有内在的排序关系（natural ordering）。为实现Comparable接口的对象数组进行排序只需如下操作：

```java
Arrays.sort(a);
```

​	实现了Comparable接口，就可以跟许多泛型算法（generic algorithm）以及依赖于该接口的集合实现（collection implement）进行协作。**事实上，Java平台类库中的所有值类（value classes），以及所有的枚举类型（E34）都实现了Comparable接口**。

​	**如果正在编写一个值类，它具有非常明显的内在排序关系，比如按字母顺序、按数值顺序或者年代顺序，那你就应该坚决考虑实现Comparable接口**：

```java
public interface Comparable<T> {
  int compareTo(T t);
}
```

---

+ compateTo方法的通用约定

  ​	与equals方法的约定相似：将这个对象与指定的对象进行比较。当该对象小于、等于或大于指定对象的时候，分别返回一个负整数、零或者正整数。如果由于指定对象的类型而无法与该对象进行比较，则抛出ClassCastException异常。

  ​	在下面的说明中，符号sgn（expression）表示数学中的signum函数，它根据表达式（expression）的值为负数、零和正值，分别返回-1、0或1。

  + 实现者必须确保所有的x和y都必须满足`sgn(x.compareTo(y)) == -sgn(y.compareTo(x))`。

    （这也暗示，当且仅当`y.compareTo(x)`抛异常时，`x.compareTo(y)`才必须抛出异常。）

  + 实现者还必须确保这个关系是可传递的：`(x.compareTo(y) > 0 && y.compateTo(z) > 0)`暗示着`x.compareTo(z) > 0`。

  + 最后，实现者必须确保`x.compareTo(y) == 0`暗示着所有的z都必须满足`sgn(x.compareTo(z)) == sgn(y.compareTo(z))`。

  + **强烈建议`(x.compareTo(y) == 0) == (x.equals(y))`，但这并非绝对必要**。

    一般来说，任何实现了Comparable接口的类，若违反了这个条件，都应该明确予以说明。建议使用这样的说法："注意：该类具有内在的排序功能，但是与equals不一致。"

  > compareTo约定的最后一段是一条强烈的建议，而不是真正的规则，它只是说明了compareTo方法施加的等同性测试，在通常情况下应该返回与equals方法同样的结果。
  >
  > 如果遵守了这一条，那么由compareTo方法所施加的顺序关系就被认为与equals一致。如果违反了这条规则，顺序关系就被认为与equals不一致。如果一个类的compareTo方法施加了一个与equals方法不一致的顺序关系，它仍然能够正常工作，但是如果一个有序集合(sorted collection)包含了该类的元素，这个集合就可能无法遵守相应集合接口(Collection、Set或Map)的通用约定。**因为对于这些接口的通用约定是按照equals方法来定义的，但是有序集合使用了由compareTo方法而不是equals方法所施加的等同性测试**。尽管出现这种情况不会造成灾难性的后果，但是应该有所了解。

  ---

  ​	千万不要被上述约定中的数学关系所迷惑。如同equa1s约定（E10）一样，compareTo约定并没有看起来那么复杂。

  ​	与equals方法不同的是，它对所有的对象强行施加了一种通用的等同关系，**compareTo不能跨越不同类型的对象进行比较**：在比较不同类型的对象时，compareto可以抛出ClassCastException异常。通常，这正是compareTo在这种情况下应该做的事情。合约确实允许进行跨类型之间的比较，这一般是在被比较对象实现的接口中进行定义。

  ​	<u>就好像违反了hashCode约定的类会破坏其他依赖于散列的类一样，违反compareTo约定的类也会破坏其他依赖于比较关系的类</u>。

  ​	**依赖于比较关系的类包括有序集合类TreeSet和TreeMap，以及工具类Collections和Arrays，它们内部包含有搜索和排序算法**。

---

+ 忠告

  ​	前面的通用约定，使得<u>由compareTo方法施加的等同性测试，也必须遵守相同于equals约定所施加的限制条件：**自反性、对称性和传递性**</u>。

  ​	因此下列的告诫也同样适用：

  + **无法在用新的值组件扩展可实例化的类时，同时保持compareTo约定，除非愿意放弃面向对象的抽象优势（E10）**。

  + 针对equals的权宜之计也同样适用于compareTo方法。**如果你想为一个实现了Comparable接口的类增加值组件，请不要扩展这个类；而是要编写一个不相关的类，其中包含第一个类的一个实例**。然后提供一个"视图"(view)方法返回这个实例。这样既可以让你自由地在第二个类上实现 compareTo方法，同时也允许它的客户端在必要的时候，把第二个类的实例视同第一个类的实例。

  + **<u>有序集合</u>使用了由compareTo方法而不是equals方法所施加的等同性测试，所以强烈建议`(x.compareTo(y) == 0) == (x.equals(y))`**。

    > ​	例如，以BigDecimal类为例，它的compareTo方法与equals不一致。如果你创x建了一个空的 Hashset实例，并且添加`new BigDecimal("1.0")`和 `new BigDecima("1.0")`，这个集合就将包含两个元素，因为<u>新增到集合中的两个BigDecimal实例，通过equals方法来比较时是不相等的</u>。
    >
    > ​	然而，如果你使用TreeSet而不是HashSet来执行同样的过程，集合中将只包含一个元素因为这两个BigDecimal实例在<u>通过compareTo方法进行比较时是相等的</u>。(详情请参阅Bigdecimal的文档)

---

+ 编写compareTo方法

  ​	编写compareTo方法与编写equals方法非常相，但也存在几处重大的差别。因为 Comparable接口是参数化的，而且comparable方法是静态的类型，因此不必进行类型检查，也不必对它的参数进行类型转换。如果参数的类型不合适，这个调用甚至无法编译。如果参数为null，这个调用应该抛出NullPointerException异常，并且一旦该方法试图访问它的成员时就应该抛出异常。

  ​	**CompareTo方法中域的比较是顺序的比较，而不是等同性的比较**。比较对象引用域可以通过递归地调用 compareTo方法来实现。<u>如果一个域并没有实现 Comparable接口，或者你需要使用一个非标准的排序关系，就可以使用一个显式的Comparator来代替</u>。或者编写自己的比较器，或者使用已有的比较器，例如针对（E10）中的CaselnsensitiveString类的这个compareTo方法使用一个已有的比较器：

  ```java
  // Single-field Comparable with object reference field
  public final class CaseInsensitiveString implements Comparable<CaseInsensitiveString> {
    public int compareTo(CaseInsensitiveString cis) {
  		return String.CASE_INSENSITIVE_ORDER.compare(s, cis.s);
    }
    ... // Remainder omitted
  }
  ```

  > 注意CaseInsensitiveString类实现了Comparable\<CaseInsensitiveString\>接口。这意味着CaseInsensitiveString引用只能与另一个CaseInsensitiveString引用进行比较。在声明类去实现Comparable接口时，这是常用的模式。

  > **在Java7版本中，Java的所有装箱基本类型的类中增加了静态的compare方法。在compareTo方法中使用关系操作符\<和\>是非常繁琐的，并且容易出错，因此不再建议使用**。

  ​	如果一个类有多个关键域，那么，按什么样的顺序来比较这些域是非常关键的。你必须从最关键的域开始，逐步进行到所有的重要域。如果某个域的比较产生了非零的结果（零代表相等），则整个比较操作结束，并返回该结果。如果最关键的域是相等的，则进一步比较次关键的域，以此类推。如果所有的域都是相等的，则对象就是相等的，并返回零。下面通过（E11）中的 PhoneNumber类的 compareTo方法来说明这种方法：

  ```java
  // Multiple-field Comaprable with primitive fields
  public int compareTo(PhoneNumber pn) {
    int result = Short.compare(areaCode, pn.areaCode);
    if(result == 0){
      result = Short.compare(prefix, pn.prefix);
      if(result == 0)
        result = Short.compare(lineNum, pn.lineNum);
    }
    return result;	
  }
  ```

  ​	<u>在Java8中，Comparator接口配置了一组比较器构造方法（comparator construction methods），使得比较器的构造工作变得非常流畅</u>。之后，按照 Comparable接口的要求，这些比较器可以用来实现一个 compareTo方法。许多程序员都喜欢这种方法的简洁性，虽然它要付出一定的性能成本：在我的机器上, PhonenNmber实例的数组排序的速度慢了大约10%。在使用这个方法时，为了简洁起见，可以考虑使用Java的静态导入(static import)设施，通过静态比较器构造方法的简单的名称就可以对它们进行引用。下面是使用这个方法之后 PhoneNumber的compareTo方法:

  ```java
  // Comparable with comparator construction methods
  private static final Comparator<PhoneNumber> COMPARATPOR = 
    comparingInt((PhoneNumber pn)-> pn.areaCode)
    .thenComparingInt(pn -> pn.prefix)
    .thenComparingInt(pn -> pn.lineNum);
  
  public int compareTo(PhoneNumber pn) {
    return COMPARATOR.compare(this, pn);
  }
  ```

  > 这个实现利用两个比较构造方法，在初始化类的时候构建了一个比较器。第一个是comparingInt。这是一个静态方法，带有一个键提取器函数(key extractor function)，它将一个对象引用映射到一个类型为int的键上，并根据这个键返回一个对实例进行排序的比较器。
  >
  > 在上一个例子中comparingInt带有一个 lambda()，它从 PhoneNumber提取区号，并返回一个按区号对电话号码进行排序的Comparator\<Phonenumber\>。注意，<u>lambda显式定义了其输入参数( Phonenumber pn)的类型</u>。事实证明，在这种情况，Java的类型推导还没有强大到足以为自己找出类型，因此我们不得不帮助它直接进行指定，以使程序能够成功地进行编译。

  > **Comparator类具备全套的构造方法**。对于基本类型long和double都有对应的comparingInt和 thenComparingInt。int版本也可以用于更狭义的整数型类型，如PhoneNumber例子中的short。 double版本也可以用于float。这样便涵盖了所有的Java数字型基本类型。

  ​	**对象引用类型也有比较器构造方法**。静态方法comparing有两个重载。一个带有键提取器，使用键的内在排序关系。第二个既带有键提取器，还带有要用在被提取的键上的比较器。这个名为thenComparing的实例方法有三个重载。一个重载只带一个比较器，并用它提供次级顺序。第二个重载只带一个键提取器，并利用键的内在排序关系作为次级顺序。最后一个重载既带有键提取器，又带有要在被提取的键上使用的比较器。

  ​	compareTo或者compare方法偶尔也会依赖于两个值之间的区别，即如果第一个值小于第二个值，则为负；如果两个值相等，则为零；如果第一个值大于第二个值，则为正。下面举个例子：

  ```java
  // BROKEN difference-based comparator - violates transitivity!
  static Comparator<Object> hashCodeOrder = new Compatator<>() {
    public int compare(Object o1, Object o2) {
      return o1.hashCode() - o2.hashCode();
    }
  }
  ```

  ​	千万不要使用这个方法，它很容易造成整数溢出，同时违反IEEE 754浮点算数标准。甚至与利用本条目讲到的方法编写的那些方法相比，最终得到的方法并没有明显变快。

  因此，要么使用一个**静态方法compare**：

  ```java
  // Comparator based on static compare method
  static Comparator<Object> hashCodeOrder = new Comparator<>() {
    public int compare(Object o1, Object o2) {
      return Integer.compare(o1.hashCode(), o2.hashCode());
    }
  };
  ```

  要么使用一个**比较构造方法**：

  ```java
  // Comparator based on Comparator construction method
  static Comparator<Object> hashCodeOrder = 
    Comparator.comparingInt(o -> o.hashCode());
  ```

---

+ 小结

  ​	总而言之，每当实现一个对排序敏感的类时，都应该让这个类实现Comparable接口，以便其实例可以轻松地被分类、搜索，以及用在基于比较的集合中。

  ​	**每当在compareTo方法的实现中比较域值时，都要避免使用\<和\>操作符，而应该在装箱基本类型的类中使用静态的compare方法，或者在Comparator接口中使用比较器构造方法**。

# 4、类和接口

## E15 使类和成员的可访问性最小化

+ 概述

  ​	**区分一个组件设计的好不好，唯一重要因素在于，它对于外部的其他组件而言，是否隐藏了其内部数据和其他实现细节**。

  ​	设计良好的组件会隐藏所有的实现细节，把API与实现清晰地隔离开来。然后，组件之间只通过API进行通信，一个模块不需要知道其他模块的内部工作情况。这个概念被称为**信息隐藏（information hiding）或封装（encapsulation）**，是软件设计的基本原则之一。

  + 信息隐藏/封装，能有效地实现各组件之间的**解耦（decouple）**。

  + 虽然信息隐藏/封装，不会带来更好的性能，但是能有效调节性能：一旦完成一个系统，并通过剖析确定了哪些组件影响了系统的性能（E67），那些组件就可以被进一步优化，而不会影响到其他组件的正确性。
  + 信息隐藏/封装提高了软件的可重用性。
  + 信息隐藏/封装降低了构建大型系统的风险，因为即使整个系统不可用，这些独立的组件仍有可能是可用的。

---

+ Java中信息隐藏的机制

  Java提供了许多机制（facility）来协助信息隐藏。

  ​	访问控制（access control）机制决定了类、接口和成员的可访问性（accessibility）。

  ​	实体的可访问性由该实体声明所在的位置，以及该实体声明中所出现的访问修饰符（private、protected和public）共同决定。

  + **尽可能使每个类或者成员不被外界访问**。

    如果一个包级私有的顶层类（或者接口）只是在某一个类的内部被用到，就应该考虑使它成为唯一使用它的那个类的私有嵌套类（E24）。这样可以将它的可访问范围从包中的所有类缩小到使用它的那个类。

  对于成员（域、方法、嵌套类和嵌套接口）有四种可能的访问级别，下面按照可访问性的递增顺序罗列出来：

  + 私有的（private）：只有在声明该成员的顶层类内部才可以访问这个成员。
  + 包级私有的（package-private）：声明该成员的包内部的任何类都可以访问这个成员。从技术上讲，它被称为"缺省（default）"访问级别，如果没有为成员指定访问修饰符，就采用这个访问级别（当然，接口成员除外，它们默认的访问级别是公有的）。
  + 受保护的（protected）：声明该成员的类的子类可以访问这个成员（但有一些限制），并且声明该成员的包内部的任何类也可以访问这个成员。
  + 公有的（public）：在任何地方都可以访问该成员。

  > 需要注意的是，类实现了Serializable接口（E86和E87），可能会导致private、protected域被"泄漏"（leak）到导出的API中。

  ​	对于公有类的成员，当访问级别从包级私有变成保护级别时，会大大增强可访问性。受保护的成员是类的导出的API的一部分，必须永远得到支持。导出的类的受保护成员也代表了该类对于某个实现细节的公开承诺（E19）。应该尽量少用受保护的成员。

  ​	有一条规则限制了降低方法的可访问性的能力。如果方法覆盖了超类中的一个方法，子类中的访问级别就不允许低于超类中的访问级别[JLS, 8.4.8.3]。这样可以确保任何可使用超类的实例的地方也都可以使用子类的实例（里氏替换原则，E10）。如果违反了这条规则，那么当你试图编译该子类的时候，编译器就会产生一条错误消息。这条规则有一个特例：如果一个类实现了一个接口，那么接口中所有的方法在这个类中也都必须被声明为公有的。

  + **公有类的实例域绝不能是公有的**（E16）。

    ​	如果实例域是非final的，或者是一个指向可变对象的final引用，那么一旦使这个域成为公有的，就等于放弃了对存储在这个域中的值进行限制的能力；这意味着，你也放弃了强制这个域不可变的能力。同时，当这个域被修改的时候，你也失去了对它采取任何行动的能力。因此，**包含公有可变域的类通常并不是线程安全的**。即使域是final的，并且引用不可变的对象，但当把这个域变成公有的时候，也就放弃了"切换到一种新的内部数据表示法"的灵活性。

  + **让类具有公有的静态final数组域，或者返回这种域的访问方法，这是错误的**。

    ​	长度非零的数组总是可变的，如果类具有这样的域或者访问方法，客户端将能修改数组中的内容，这是安全漏洞的一个常见根源：

    ```java
    // Potential security hole!
    public static final Thing[] VALUES = { ... };
    ```

    > 注意，许多IDE产生的访问方法会返回指向私有数组域的引用，正好导致这个问题。修正这个问题有两种方法。
    >
    > + **使公有数组变成私有的，并增加一个公有的不可变列表**：
    >
    > ```java
    > private static final Thing[] PRIVATE_VALUES = { ... };
    > public static final List<Thing> VALUES = 
    >   Collections.unmodifiableList(Arrays.asList(PRIVATE_VALUES));
    > ```
    >
    > + **使数组变成私有的，并增加一个公有方法，它返回私有数组的一个拷贝**：
    >
    > ```java
    > private static final Thing[] PRIVATE_VALUES = { ... };
    > public static final Thing[] values() {
    >   return PRIVATE_VALUES.clone();
    > }
    > ```

  ​	<u>从Java9开始，又新增了两种隐式访问级别，作为模块系统（module system）的一部分。一个模块就是一组包，就像一个包就是一组类一样。模块可以通过其模块声明（module declaration）中的导出声明（export declaration）显式地导出它的一部分包（按照惯例，这包含在名为module-info.java的源文件中）。模块中未被导出的包在模块之外是不可访问的；在模块内部，可访冋性不受导岀声明的影响</u>。使用模块系统可以在模块内部的包之间共享类，不用让它们对全世界都可见。未导出的包中公有类的公有成员和受保护的成员都提高了两个隐式访问级别，这是正常的公有和受保护级别在模块内部的对等体（ intramodularanalogues)。对于这种共享的需求相对罕见，经常通过在包内部重新安排类来解决。

  ​	与四个主访问级别不同的是，这两个基于模块的级别主要提供咨询。如果把模块的JAR文件放在应用程序的类路径下，而不是放在模块路径下，模块中的包就会恢复其非模块的行为：无论包是否通过模块导出，这些包中公有类的所有公有的和受保护的成员将都有正常的可访问性[Reinhold,1.2]。<u>严格执行新引入的访问级别的一个示例是JDK本身：Java类库中未导出的包在其模块之外确实是不可访问的</u>。

  ​	<small>对于传统的Java程序员来说，不仅由受限工具的模块提供了访问保护，而且在本质上主要也是提供咨询。为了利用模块的这一特性，必须将包集中到模块中，并在模块声明中显式地表明其所有的依赖关系，重新安排代码结构树，从模块内部采取特殊的动作调解对于非模块化的包的任何访问[ Reinhold,3]。**现在说模块将在JDK本身之外获得广泛的使用，还为时过早。同时，似乎最好不用它们,除非你的需求非常迫切**。</small>

---

+ 小结

​	**总而言之，应该始终尽可能(合理)地降低程序元素的可访问性**。在仔细地设计了一个最小的公有API之后，应该防止把任何散乱的类、接口或者成员变成API的一部分。除了公有静态final域的特殊情形之外（此时它们充当常量），**公有类都不应该包含公有域，并且要确保公有静态final域所引用的对象都是不可变的**。

## E16 要在公有类而非公有域中使用访问方法

+ 概述

​	毫无疑问，说到公有类的时候，坚持面向对象编程思想的看法是正确的：**如果类可以在它所在的包之外进行访问，就提供访问方法**，以保留将来改变该类的内部表示法的灵活性。如果公有类暴露了它的数据域，要想在将来改变其内部表示法是不可能的，因为公有类的客户端代码已经遍布各处了。

​	然而，**如果类是包级私有的，或者是私有的嵌套类，直接暴露它的数据域并没有本质的错误**——假设这些数据域确实描述了该类所提供的抽象。无论是在类定义中，还是在使用该类的客户端代码中，这种方法比访问方法的做法更不容易产生视觉混乱。<u>虽然客户端代码与该类的内部表示法紧密相连，但是这些代码被限定在包含该类的包中</u>。如有必要，也可以不改变包之外的任何代码，而只改变内部数据表示法。在私有嵌套类的情况下，改变的作用范围被进一步限制在外围类中。

​	<u>Java平台类库中有几个类违反了"公有类不应该直接暴露数据域"的告诫。显著的例子包括java.awt包中的Point类和Dimension类。它们是不值得仿效的例子，相反，这些类应该被当作反面的警告示例</u>。正如（E67）所述，决定暴露Dimension类的内部数据造成了严重的性能问题，而且这个问题至今依然存在。

​	让公有类直接暴露域虽然从来都不是种好办法，但是如果域是不可变的，这种做法的危害就比较小一些。如果不改变类的API，就无法改变这种类的表示法，当域被读取的时候，你也无法采取辅助的行动，但是可以强加约束条件。例如，这个类确保了每个实例都表示一个有效的时间：

```java
// Public class with expoesd immutable fields - questionable
public final class Time {
  private static final int HOURS_PER_DAY = 24;
  private static final int MINUTES_PER_HOUR = 60;

  public final int hour;
  public final int minute;

  public Time(int hour, int minute) {
    if(hour < 0 || hour >= HOURS_PER_DAY)
      throw new IllegalArgumentException("Hour: " + hour);
    if(minute < 0 || minute >= MINUTES_PER_HOUR)
      throw new IllegalArgumentException("Min: " + hour);
    this.hour = hour;
    this.minute = minute;
  }
  ... // Remainder omitted
}  
```

----

+ 小结

​	简而言之，公有类永远都不应该暴露可变的域。虽然还是有问题，但是让公有类暴露不可变的域，其危害相对来说比较小。但有时候会需要用包级私有的或者私有的嵌套类来暴露域，无论这个类是可变的还是不可变的。

## * E17 使可变性最小化

+ 概述

​	<u>不可变类是指其实例不能被修改的类。每个实例中包含的所有信息都必须在**创建该实例的时候就提供**，并在对象的整个生命周期（lifetime）内固定不变</u>。

​	Java平台类库中包含许多不可变的类，其中有String、基本类型的包装类、 BigInteger和BigDecimal。

​	存在不可变的类有许多理由：**不可变的类比可变类更加易于设计、实现和使用。它们不容易出错，且更加安全**。

​	为了使类成为不可变，要遵循下面五条规则：

1. **不要提供任何会修改对象状态的方法**（也称为设值方法）。

2. **保证类不会被扩展**。这样可以防止粗心或者恶意的子类假装对象的状态已经改变，从而破坏该类的不可变行为。为了防止子类化，一般做法是声明这个类成为final的，但是后面我们还会讨论到其他的做法。

3. **声明所有的域都是final的**。通过系统的强制方式可以清楚地表明你的意图。而且，如果一个指向新创建实例的引用在缺乏同步机制的情况下，从一个线程被传递到另一个线程，就必须确保正确的行为，正如内存模型(memory model)中所述\[JLS,17.5; Goetz06 16\]。

4. **声明所有的域都为私有的**。这样可以防止客户端获得访问被域引用的可变对象的权限，并防止客户端直接修改这些对象。虽然从技术上讲，允许不可变的类具有公有的fnal域，只要这些域包含基本类型的值或者指向不可变对象的引用，但是不建议这样做，因为这样会使得在以后的版本中无法再改变内部的表示法（E15和E16）。
5. **确保对于任何可变组件的互斥访问**。如果类具有指向可变对象的域，则必须确保该类的客户端无法获得指向这些对象的引用。并且，永远不要用客户端提供的对象引用来初始化这样的域，也不要从任何访问方法（accessor）中返回该对象引用。在构造器、访问方法和readObject方法（E88）中请使用保护性拷贝（defensive copy）技术（E50）。

---

不可变对象的特性：

+ **不可变对象比较简单**。

  ​	<small>不可变对象可以只有一种状态，即被创建时的状态。如果你能够确保所有的构造器都建立了这个类的约束关系，就可以确保这些约束关系在整个生命周期内永远不再发生变化，你和使用这个类的程序员都无须再做额外的工作来维护这些约束关系。另一方面，可变的对象可以有任意复杂的状态空间。如果文档中没有为设值方法所执行的状态转换提供精确的描述，要可靠地使用可变类是非常困难的，甚至是不可能的。</small>

+ **不可变对象本质上是线程安全的，它们不要求同步**。

  ​	当多个线程并发访问这样的对象时，它们不会遭到破坏。这无疑是获得线程安全最容易的办法。实际上，没有任何线程会注意到其他线程对于不可变对象的影响。所以，**不可变对象可以被自由地共享**。不可变类应该充分利用这种优势，鼓励客户端尽可能地重用现有的实例。<u>要做到这一点，一个很简便的办法就是：对于频繁用到的值，为它们提供公有的静态final常量。</u>

  ​	不可变的类可以提供一些静态工厂（E1），它们把频繁被请求的实例缓存起来，从而当现有实例可以符合请求的时候，就不必创建新的实例。<u>所有基本类型的包装类和BigInteger都有这样的静态工厂</u>。使用这样的静态工厂也使得客户端之间可以共享现有的实例,而不用创建新的实例，从而降低内存占用和垃圾回收的成本。在设计新的类时，选择用静态工厂代替公有的构造器可以让你以后有添加缓存的灵活性，而不必影响客户端。

  ​	<u>"不可变对象可以被自由地共享"导致的结果是，永远也不需要进行保护性拷贝(defensive copy)（E50）。实际上，你根本无须做任何拷贝，因为这些拷贝始终等于原始的对象</u>。因此，你不需要，也不应该为不可变的类提供clone方法或者拷贝构造器（E13）。这一点在Java平台的早期并不好理解，所以String类仍然具有拷贝构造器，但是应该尽量少用它（E6）

+ **不仅可以共享不可变对象，甚至也可以共享它们的内部信息**。

  ​	例如，BigInteger类内部使用了符号数值表示法。符号用一个int类型的值来表示，数值则用一个int数组表示。negate方法产生一个新的BigInteger，其中数值是一样的，符号则是相反的。它并不需要拷贝数组，新建的BigInteger也指向原始实例中的同一个内部数组。

+ **不可变对象为其他对象提供了大量的构件**，无论是可变的还是不可变的对象。

  ​	如果知道一个复杂对象内部的组件对象不会改变，要维护它的不变性约束是比较容易的。这条原则的一种特例在于，不可变对象构成了大量的映射键（map key）和集合元素（set element）；一旦不可变对象进入到映射(map)或者集合(set)中，尽管这破坏了映射或者集合的不变性约東，但是也不用担心它们的值会发生变化。

+ **不可变对象无偿地提供了失败的原子性**（E76）。

  ​	它们的状态永远不变，因此不存在临时不一致的可能性。

+ **不可变类真正唯一的缺点是，对于每个不同的值都需要一个单独的对象**。

  ​	创建这些对象的代价可能很高，特别是大型的对象。例如，假设你有一个上百万位的BigInteger，想要改变它的低位：

  ```java
  BigInteger moby = ...;
  moby = moby.flipBit(0);
  ```

> ​	如果能够精确地预测出客户端将要在不可变的类上执行哪些复杂的多阶段操作，这种包级私有的可变配套类的方法就可以工作得很好。如果无法预测，最好的办法是提供一个公有的可变配套类。在Java平台类库中，这种方法的主要例子是String类，它的可变配套类是StringBuilder（及其已经被废弃的祖先StringBuffer）。

---

+ 构造不可变的类的其他方案

​	现在你已经知道了如何构建不可变的类，并且了解了不可变性的优点和缺点，现在我们来讨论其他的一些设计方案。前面提到过，为了确保不可变性，类绝对不允许自身被子类化。

​	**除了"使类成为fina的"这种方法之外，还有另外一种更加灵活的办法可以做到这点。不可变的类变成final的另一种办法就是，让类的所有构造器都变成私有的或者包级私有的，并添加公有的静态工厂(static factory)来代替公有的构造器(E1)**。

> ​	当 BigInteger和BigDecimal刚被编写出来的时候，对于"不可变的类必须为final"的说法还没有得到广泛的理解，所以它们的所有方法都有可能被覆盖。遗憾的是，为了保持向后兼容，这个问题一直无法得以修正。如果你在编写一个类，它的安全性依赖于来自不可信客户端的BigInteger或者Bigdecimal参数的不可变性，就必须进行检查，以确定这个参数是否为"真正的"BigInteger或者Bigdecimal，而不是不可信任子类的实例。如果是后者，就必须在假设它可能是可变的前提下对它进行保护性拷贝（E50）
>
> ```java
> public static BigInteger safeInstance(BigInteger val) {
>   return val.getClass() == BigInteger.class ? val : new BigInteger(val.toByteArray());
> }
> ```

---

+ 有关序列化功能的告诫

​	如果你选择让自己的**不可变类实现Serializable接口**，并且它包含一个或者多个指向可变对象的域，必须**提供一个显式的readObject或者readResolve方法，或者使用`ObjectOutputStream.writeUnshared`和 `ObjectInputStream.readUnshared`方法**，即便默认的序列化形式是可以接受的，也是如此。否则，<u>攻击者可能从不可变的类创建可变的实例</u>。关于这个话题的详情请参见（E88）

----

+ 小结

​	总之，坚决不要为每个get方法编写一个相应的set方法。**除非有很好的理由要让类成为可变的类，否则它就应该是不可变的**。不可变的类有许多优点，唯一的缺点是在特定的情况下存在潜在的性能问题。你应该总是使一些小的值对象，比如Phonenumber和Complex，成为不可变的。(在Java平台类库中，有几个类如`java.util.Date`和`java.awt.Point`，它们本应该是不可变的，但实际上却不是。)你也应该认真考虑把一些较大的值对象做成不可变的，例如String和BigInteger。只有当你确认有必要实现令人满意的性能时（E67），才应该为不可变的类提供公有的可变配套类。

​	对于某些类而言，其不可变性是不切实际的。**如果类不能被做成不可变的，仍然应该尽可能地限制它的可变性**。降低对象可以存在的状态数，可以更容易地分析该对象的行为，同时降低出错的可能性。因此，除非有令人信服的理由使域变成非final的，否则让每个域都是final的。结合这条的建议和（E15）的建议，你自然倾向于：**除非有令人信服的理由要使域变成是非final的，否则要使每个域都是private final的**。

​	**构造器应该创建完全初始化的对象，并建立起所有的约束关系**。不要在构造器或者静态工厂之外再提供公有的初始化方法，除非有令人信服的理由必须这么做。同样地，也不应该提供"重新初始化"法(它使得对象可以被重用，就好像这个对象是由另一不同的初始状态构造出来的一样)。与所增加的复杂性相比，"重新初始化"方法通常并没有带来太多的性能优势。

​	<u>通过CountDownLatch类的例子可以说明这些原则。它是可变的，但是它的状态空间被有意地设计得非常小。比如创建一个实例，只使用一次，它的任务就完成了：一旦定时器的计数达到零，就不能重用了</u>。

> ​	最后值得注意的一点与本条目中的Complex类有关。这个例子只是被用来演示不可变性的，它不是一个工业强度的复数实现。它对复数乘法和除法使用标准的计算公式，会进行不正确的四舍五入，并且对复数NaN和无穷大也没有提供很好的语义[Kahan91，Smith62，Thomas94]。
> ​	ps：此处笔记没有摘录这部分代码，想了解可以看原书。

## * E18 复合优先于继承

+ 概述

  ​	继承（inheritance）是实现代码重用的有力手段，但使用不当会导致软件变得脆弱。

  + **与方法调用不同的是，继承打破了封装性**。

    ​	换句话说，子类依赖于其超类中特定功能的实现细节。超类的实现有可能会随着发行版本的不同而有所变化，如果真的发生了变化，子类可能会遭到破坏，即使它的代码完全没有改变。因而，子类必须要跟着其超类的更新而演变，除非超类是专门为了扩展而设计的，并且具有很好的文档说明。

  + 继承使得子类变得"脆弱"（"覆盖"导致的问题）

    ​	子类无感知超类的升级，超类有可能后续提供了一个签名和子类相同但是返回类型不同的方法，导致子类无法通过编译。或者子类无意间**覆盖**了超类的方法，但是不能保证遵守了超类方法的约定（因为这个被覆盖的方法是超类后续才添加上的）。

---

+ 避免"覆盖"产生的混乱场景——"复合"（composition）

  ​	避免"覆盖"导致的复杂情况，**不扩展现有的类，而是在新的类中增加一个私有域，它引用现有类的一个实例。这种设计被称为"复合"（composition）**。

  ​	**新类中的每个实例方法都可以调用被包含的现有类实例中对应的方法，并返回它的结果。这被称为转发( forwarding)，新类中的方法被称为转发方法(forwarding method)**。这样得到的类将会非常稳固，它不依赖于现有类的实现细节。即使现有的类添加了新的方法，也不会影响新的类。

  ​	为了进行更具体的说明，请看下面的例子，它用复合/转发的方法来代替InstrumentedHashSet类。注意这个实现分为两部分：**类本身和可重用的转发类(forwarding class)**，其中包含了所有的转发方法，没有任何其他的方法：

  ```java
  // Wrapper class - uses composition in place of inheritance
  public class InstrumentedSet<E> extends ForwardingSet<E> {
    private int addCount = 0;
    
    public InstrumentedSet(Set<E> s) {
      super(s);
    }
    
    @Override public boolean add(E e) {
      addCount++;
      return super.add(e);
    }
    
    @Override public boolean addAll(Collection<? extends E> c) {
  		addCount += c.size();
      return super.addAll(c);
    }
    
    public int getAddCount() {
      return addCount;
    }
  }
  
  // Reusable forwading class
  public class ForwardingSet<E> implements Set<E> {
    private final Set<E> s;
    public ForwardingSet(Set<E> s) { this.s = s; }
    
    public void clear() { s.clear(); }
    public boolean contains(Object o) { return s.contains(o); }
    public boolean isEmpty() { return s.isEmpty(); }
    public int size() { return s.size(); }
    public Iterator<E> iterator() { return s.iterator(); }
    public boolean add(E e) { return s.add(e); }
    public boolean remove(Object o) { return s.remove(o); }
    public boolean containsAll(Collection<?> c) { return s.containsAll(c); }
    public boolean addAll(Collectioin<?> c) { return s.addAll(c); }
    public boolean removeAll(Collection<?> c) { return s.removeAll(c); }
    public boolean retainAll(Collection<?> c) { return s.retainAll(c); }
    public Object[] toArray() { return s.toArray(); }
  	public <T> T[] toArray(T[] a) { return s.toArray(a); }
    @Override public boolean equals(Object o) { return s.equals(o); }
    @Override public int hashCode() { return s.hashCode(); }
    @Override public String toString() { return s.toString(); }
  }
  ```

  ​	Set接口的存在使得InstrumentedSet类的设计成为可能，因为Set接口保存了HashSet类的功能特性。除了获得健壮性之外，这种设计也带来了更多的灵活性。InstrumentedSet类实现了Set接口，并且拥有单个构造器，它的参数也是Set类型。从本质上讲，这个类把一个Set转变成了另一个Set，同时增加了计数的功能。

  ​	<u>前面提到的基于继承的方法只适用于单个具体的类，并且对于超类中所支持的每个构造器都要求有个单独的构造器</u>，与此不同的是，这里的包装类(wrapper class)可以被用来包装任何Set实现，并且可以结合任何先前存在的构造器一起工作：

  ```java
  Set<Instant> times = new InstrumentedSet<>(new TreeSet<>(cmp));
  Set<E> s = new InstrumentedSet<>(new HashSet<>(INIT_CAPACITY));
  ```

  ​	InstrumentedSet类甚至也可以用来临时替换一个原本没有计数特性的Set实例

  ```java
  static void walk(Set<Dog> dogs) {
    InstrumentedSet<Dog> iDogs = new InstrumentedSet<>(dogs);
    ... // Within this method use iDogs instead of dogs
  }
  ```

  ​	因为每一个InstrumentedSet实例都把另一个Set实例包装起来了，所以InstrumentedSet类被称为包装类( wrapper class)。这也正是**Decorator(修饰者)模式**，因为 InstrumentedSet类对一个集合进行了修饰，为它增加了计数特性。<u>有时复合和转发的结合也被宽松地称为"委托"(delegation)。从技术的角度而言，这不是委托，除非包装对象把自身传递给被包装的对象</u>。

---

+ 包装类不适合用于回调框架（callback framework）

  ​	在回调框架中，对象把自身的引用传递给其他的对象，用于后续的调用("回调")。<u>因为被包装起来的对象并不知道它外面的包装对象，所以它传递一个指向自身的引用(this)，回调时避开了外面的包装对象。这被称为SELF问题</u>。

  ​	**有些人担心转发方法调用所带来的性能影响，或者包装对象导致的内存占用。在实践中，这两者都不会造成很大的影响**。编写转发方法倒是有点琐碎，但是只需要给每个接口编写一次构造器，转发类则可以通过包含接口的包提供。例如，Guava就为所有的集合接口提供了转发类Guava。

---

- 何时使用继承

  + **只有当子类真正是超类的子类型(subtype)时，才适合用继承**。

    ​	<small>换句话说，对于两个类A和B，只有当两者之间确实存在“is-a”关系的时侯，类B才应该扩展类A。如果你打算让类B扩展类A，就应该问问自己：每个B确实也是A吗?如果你不能够确定这个问题的答案是肯定的，那么B就不应该扩展A。如果答案是否定的，通常情况下，B应该包含A的一个私有实例，并且暴露一个较小的、较简单的API：A本质上不是B的一部分，只是它的实现细节而已。</small>

    > ​	在Java平台类库中,有许多明显违反这条原则的地方。例如，栈(Stack)并不是向量(Vector)，所以Stack不应该扩展 Vector。同样地，属性列表也不是散列表，所以Properties不应该扩展Hashtable。在这两种情况下，复合模式才是恰当的。

  + **如果在适合使用复合的地方使用了继承，则会不必要地暴露实现细节**。

    ​	<u>这样得到的API会把你限制在原始的实现上，永远限定了类的性能</u>。更为严重的是，由于暴露了内部的细节，客户端就有可能直接访问这些內部细节。这样至少会导致语义上的混淆。

    ​	例如，如果p指向Properties实例，那么`p.getProperty(key)`就有可能产生与`p.get(key)`不同的结果：前一个方法考虑了默认的属性表，而后一个方法则继承自Hashtable，没有考虑默认的属性列表。

    ​	<u>最严重的是，客户有可能直接修改超类，从而破坏子类的约束条件</u>。在Properties的情形中，设计者的目标是只允许字符串作为键(key)和值(value)，但是直接访问底层的Hashtable就允许违反这种约束条件。一旦违反了约東条件，就不可能再使用Properties API的其他部分(load和store)了。等到发现这个问题时，要改正它已经太晚了，因为客户端依赖于使用非字符串的键和值了。

---

+ 小结

  ​	在决定使用继承而不是复合之前，还应该问自己最后一组问题。对于你正试图扩展的类，它的API中有没有缺陷呢？如果有，你是否愿意把那些缺陷传播到类的APⅠ中？继承机制会把超类API中的所有缺陷传播到子类中，而复合则允许设计新的API来隐藏这些缺陷。

  ​	简而言之，**继承的功能非常强大，但是也存在诸多问题，因为它违背了封装原则**。

  ​	**只有当子类和超类之间确实存在子类型关系时，使用继承才是恰当的**。

  ​	即便如此，**如果子类和超类处在不同的包中，并且超类并不是为了继承而设计的，那么继承将会导致脆弱性(fragility)。为了避免这种脆弱性，可以用复合和转发机制来代替继承，尤其是当存在适当的接口可以实现包装类的时候。包装类不仅比子类更加健壮，而且功能也更加强大**。

## E19 要么设计继承而提供文档说明，要么禁止继承

+ 概述

  + **为了继承而设计的类，必须有文档说明它可覆盖(overridable)的方法的自用性(self-use)**。

    ​	对于每个公有的或受保护的方法或者构造器，它的文档必须指明该方法或者构造器调用了哪些可覆盖的方法，是以什么顺序调用的，每个调用的结果又是如何影响后续处理过程的(所谓可覆盖(overridable)的方法，是指非 final的、公有的或受保护的)。

    ​	更广义地说，即类必须在文档中说明，在哪些情况下它会调用可覆盖的方法。例如，后台的线程或者静态的初始化器(initializer)可能会调用这样的方法。

    > ​	关于程序文档有句格言：**好的API文档应该描述一个给定的方法做了什么工作，而不是描述它是如何做到的**。为了设计一个类的文档，以便它能够被安全地子类化，你必须描述清楚那些有可能未定义的实现细节。

    > @implSpec标签是在Java8中增加的，在Java9中得到了广泛应用。

  + **为了继承而设计的类必须以精心挑选的受保护的(protected)方法的形式，提供适当的钩子(hook)，以便进入其内部工作中**。

    ​	这一个只能纯靠经验和实验，来判断该暴露哪些protected的方法，无捷径。

  + **对于为了继承而设计的类，唯一的测试方法就是编写子类**。

    ​	经验表明，3个子类通常就足以测试一个可扩展的类。除了超类的程序设计者以外，都需要编写一个或者多个这种子类。

  + **必须在发布类之前先编写子类对类进行测试。**

    ​	在为了继承而设计有可能被广泛使用的类时，必须要意识到，对于文档中所说明的自用模式(self-use pattern)，以及对于其受保护方法和域中所隐含的实现策略，你实际上已经做出了永久的承诺。这些承诺使得你在后续的版本中提高这个类的性能或者增加新功能都变得非常困难，甚至不可能。

  + **为了允许继承，必须保证构造器绝不能调用可被覆盖的方法**。

    ​	无论是直接调用还是间接调用。如果违反了这条规则，很有可能导致程序失败。超类的构造器在子类的构造器之前运行，所以，子类中覆盖版本的方法将会在子类的构造器运行之前先被调用。如果该覆盖版本的方法依赖于子类构造器所执行的任何初始化工作，该方法将不会如预期般执行。为了更加直观地说明这一点，下面举个例子，其中有个类违反了这条规则：

    ```java
    public class Super {
      // Broken - constructor invokes an overridable method
      public Super() {
        overrideMe();
      }
      public void overrideMe() {
      }
    }
    ```

    ​	下面的子类覆盖了方法overrideMe，Super唯一的构造器就错误地调用了这个方法：

    ```java
    public final class Sub extends Super {
      // Blank final, set by constructor
      private final Instant instant;
      
      Sub() {
        instant = Instant.now();
      }
      
      // Overriding method invoked by superclass constructor
      @Override public void overrideMe() {
        System.out.println(instant);
      }
      
      public static void main(String[] args) {
        Sub sub = new Sub();
        sub.overrideMe();
      }
    }
    ```

    ​	<u>你可能会期待这个程序会打印两次日期，但是它第一次打印出的是null，因为overrideMe方法被 Super构造器调用的时候，构造器Sub还没有机会初始化instant域</u>。

    ​	注意，这个程序观察到的final域处于两种不同的状态。还要注意，如果overrideMe已经调用了 instant中的任何方法，当 Super构造器调用overrideMe的时候，调用就会抛出NullPointerException异常。如果该程序没有抛出NullPointerException异常，唯一的原因就在于println方法可以容忍null参数。

  + **注意，通过构造器调用私有的方法、 final方法和静态方法是安全的，这些都不是可以被覆盖的方法**。

---

+ 为了继承而设计类，与Cloneable和Serializable接口

  ​	如果你决定在一个为了继承而设计的类中实现Cloneable或者Serializable接口，就应该意识到，因为clone和readObject方法在行为上非常类似于构造器，所以类似的限制规则也是适用的：**无论是clone还是 readObject，都不可以调用可覆盖的方法。不管是以直接还是间接的方式**。

  + 对于readObject方法，覆盖的方法将在子类的状态被反序列化(deserialized)之前先被运行；
  + 而对于clone方法，覆盖的方法则是在子类的clone方法有机会修正被克隆对象的状态之前先被运行。

  ​	无论哪种情形，都不可避免地将导致程序失败。在clone方法的情形中，这种失败可能会同时损害到原始的对象以及被克隆的对象本身。例如，如果覆盖版本的方法假设它正在修改对象深层结构的克隆对象的备份，就会发生这种情况，但是该备份还没有完成。

  ​	**最后，如果你决定在一个为了继承而设计的类中实现Serializable接口，并且该类有一个readResolve或者 writeReplace方法，就必须使readResolve或者writeReplace成为受保护的方法，而不是私有的方法**。如果这些方法是私有的，那么子类将会不声不响地忽略掉这两个方法。这正是"为了允许继承，而把实现细节变成一个类的API的一部分"的另一种情形。	

---

+ 禁止子类化的方法

  ​	**对于那些并非为了安全地进行子类化而设计和编写文档的类，要禁止子类化**。有两种方法可以禁止子类化：

  + 类声明为final的。

  + 把所有构造器变成私有的，或者包级私有的，并增加一些公有的静态工厂来替代构造器。

    （这种方法在E17中讨论过，为内部使用子类提供了灵活性。）

  ​	如果类实现了某个能够反映其本质的接口，比如Set、List或者Map，就不应该为了禁止子类化而感到后悔。（E18）中介绍的<u>包装类（wrapper class）模式</u>还提供了另一种更好的方法，让继承机制实现更多的功能。

  ​	如果具体的类没有实现标准的接口，那么禁止继承可能会给某些程序员带来不便。如果你认为必须允许从这样的类继承，一种合理的办法是确保这个类永远不会调用它的任何可覆盖的方法，并在文档中说明这一点。换句话说，完全消除这个类中可覆盖方法的自用特性。这样做之后，就可以创建"能够安全地进行子类化"的类。覆盖方法将永远不会影响其他任何方法的行为。

  ​	你可以机械地消除类中可覆盖方法的自用特性，而不改变它的行为。将每个可覆盖方法的代码体移到一个私有的"辅助方法"(helper method)中，并且让每个可覆盖的方法调用它的私有辅助方法。然后用"直接调用可覆盖方法的私有辅助方法"来代替"可覆盖方法的每个自用调用"。

---

+ 小结

  ​	简而言之，专门为了继承而设计类是一件很辛苦的工作。你必须建立文档说明其所有的自用模式，并且一旦建立了文档，在这个类的整个生命周期中都必须遵守。**如果没有做到，子类就会依赖超类的实现细节，如果超类的实现发生了变化，它就有可能遭到破坏**。

  ​	为了允许其他人能编写出高效的子类，你必须导出一个或者多个受保护的方法。

  ​	**除非知道真正需要子类，否则最好通过将类声明为final，或者确保没有可访问的构造器来禁止类被继承**。

## * E20 接口优于抽象类

+ 概述

  Java提供了两种机制，可以用来定义允许多个实现的类型：接口和抽象类。

  自从Java8为继承引入了缺省方法(default method)，这两种机制都允许为某些实例方法提供实现。

  + **现有的类可以很容易被更新，以实现新的接口**。

    ​	如果这些方法尚不存在，你所需要做的就只是增加必要的方法，然后在类的声明中增加一个 implements子句。

    ​	例如，当Comparable、Iterable和Autocloseable接口被引入Java平台时，更新了许多现有的类，以实现这些接口。**一般来说，无法更新现有的类来扩展新的抽象类**。如果你希望两个类扩展同一个抽象类，就必须把抽象类放到类型层次(type hierarchy)的高处，这样它就成了那两个类的一个祖先。遗憾的是，这样做会间接地伤害到类层次，迫便这个公共祖先的所有后代类都扩展这个新的抽象类，无论它对于这些后代类是否合适。

  + **接口是定义mixin(混合类型)的理想选择**。

    ​	不严格地讲，mixin类型是指：类除了实现它的"基本类型"之外，还可以实现这个mixin类型，以表明它提供了某些可供选择的行为。

    ​	例如，Comparable是一个mixin接口，它允许类表明它的实例可以与其他的可相互比较的对象进行排序。这样的接口之所以被称为mixin，是因为它允许任选的功能可被混合到类型的主要功能中。**抽象类不能被用于定义mixin，同样也是因为它们不能被更新到现有的类中：类不可能有一个以上的父类，类层次结构中也没有适当的地方来插入mixin。**

  + **接口允许构造非层次结构的类型框架**。

  + **接口使得安全地增强类的功能称为可能**。

    ​	这点在（E18）中介绍的包装类（wrapper class）模式中已有描述。如果使用抽象类来定义类型，那么除了使用继承的手段来增加功能，再没有其他的选择了。继承得到的累与包装类相比，功能更差，也更加脆弱。

---

+ 接口的缺省方法

  ​	当一个接口方法根据其他接口方法有了明显的实现时，可以考虑以缺省方法的形式为程序员提供实现协助。关于这种方法的范例，请参考（E21）中的removeIf方法。如果提供了缺省方法，要确保利用Javadoc标签@implSpec建立文档（E19）。

  ​	通过缺省方法可以提供的实现协助是有限的。虽然许多接口都定义了Object方法的行为，如equals和 hashCode，但是不允许给它们提供缺省方法。而且接口中不允许包含实例域或者非公有的静态成员(私有的静态方法除外)。最后一点，无法给不受你控制的接口添加缺省方法。

---

+ 骨架实现类——结合接口和抽象类的优点。

  ​	通过对接口提供一个抽象的骨架实现（skeletal implementation）类，可以把接口和抽象类的优点结合起来。接口负责定义类型，或许还提供一些缺省方法，而骨架实现类则负责实现除基本类型接口方法之外，剩下的非基本类型接口方法。扩展骨架实现占了实现接口之外的大部分工作。这就是**模板方法(Template Method)模式**。

  ​	<u>按照惯例，骨架实现类被称为AbstractInterface，这里的Interface是指所实现的接口的名字</u>。例如， Collections Framework为每个重要的集合接口都提供了一个骨架实现，包括AbstractCollection、 AbstractSet、 AbstractList和AbstractMap。将它们称作SkeletalCollection、SkeletalSet、SkeletalList和SkeletalMap也是有道理的，但是现在Abstract的用法已经根深蒂固。

  ​	如果设计得当，骨架实现(无论是单独一个抽象类，还是接口中唯一包含的缺省方法)可以使程序员非常容易地提供他们自己的接口实现。例如，下面是一个静态工厂方法，除AbstractList之外，它还包含了一个完整的、功能全面的List实现：

  ```java
  // Concrete implementation built atop skeletal impelementation
  static List<Integer> intArrayAsList(int[] a) {
    Object.requireNonNull(a);
    
    // The diamond operator is only legal here in Java 9 and later
    // If you're using an earlier release, specify <Integer>
    return new AbstractList<> {
      @Override public Integer get(int i) {
        return a[i]; // Autoboxing (Item 6)
      }
      @Override public Integer set(int i, Integer val) {
        int oldVal = a[i];
        a[i] = val; // Auto-unboxing
        return oldVal; // Autoboxing
      }
      @Override public int size() {
        return a.length;
      }
    };
  }
  ```

  ​	如果想知道一个List实现应该为你完成哪些工作，这个例子就充分演示了骨架实现的强大功能。顺便提一下，这个例子是个**Adapter**，它允许将int数组看作Integer实例的列表。由于在int值和Integer实例之间来回转换需要开销，它的性能不会很好。注意，这个实现采用了**匿名类(anonymous class)**的形式（E24）。

  ​	骨架实现类的美妙之处在于，它们为抽象类提供了实现上的帮助，但又不强加"抽象类被用作类型定义时"所特有的严格限制。对于接口的大多数实现来讲，扩展骨架实现类是个很显然的选择，但并不是必须的。如果预置的类无法扩展骨架实现类，这个类始终能手工实现这个接口。同时，这个类本身仍然受益于接口中出现的任何缺省方法。

  ​	此外，骨架实现类仍然有助于接口的实现。<u>实现了这个接口的类可以把对于接口方法的调用转发到个内部私有类的实例上，这个内部私有类扩展了骨架实现类</u>。这种方法被称作**模拟多重继承（simulated multiple inheritance）**，它与（E18）中讨论过的包装类模式密切相关。这项技术具有多重继承的绝大多数优点，同时又避免了相应的缺陷。

  > ​	因为骨架实现类是为了继承的目的而设计的，所以应该遵从（E19）中介绍的所有关于设计和文档的指导原则。对于骨架实现类而言，好的文档绝对是非常必要的，无论它是否在接口或者单独的抽象类中包含了缺省方法。

  ​	骨架实现上有个小小的不同，就是**简单实现(simple implementation)**，`AbstractMap.SimpleEntry`就是个例子。简单实现就像骨架实现一样，这是因为它实现了接口，并且是为了继承而设计的，但是区别在于它不是抽象的：它是最简单的可能的有效实现。你可以原封不动地使用，也可以看情况将它子类化。

---

+ 小结

  ​	总而言之，**接口通常是定义允许多个实现的类型的最佳途径**。

  ​	**如果你导出了一个重要的接口，就应该坚决考虑同时提供骨架实现类**。而且，还应该尽可能地通过缺省方法在接口中提供骨架实现，以便接口的所有实现类都能使用。也就是说，对于接口的限制，通常也限制了骨架实现会采用的抽象类的形式。

## E21 为后代设计接口

+ 概述

  ​	在Java8发行之前，如果不破坏现有的实现，是不可能给接口添加方法的。如果给某个接口添加了一个新的方法，一般来说，现有的实现中是没有这个方法的，因此就会导致编译错误。

  ​	**在Java8中，增加了缺省方法(default method)构造[JLS 9.4]，目的就是允许给现有的接口添加方法**。但是给现有接口添加新方法还是充满风险的。

  ​	**Java8在核心集合接口中增加了许多新的缺省方法，主要是为了便于使用lambda(详见第6章)**。Java类库的缺省方法是高品质的通用实现，它们在大多数情况下都能正常使用。但是，并非每一个可能的实现的所有变体，始终都可以编写出一个缺省方法。

  ​	比如，以removeIf方法为例，它在Java8中被添加到了Collection接口。这个方法用来移除所有元素，并用一个boolean函数(或者断言)返回true。缺省实现指定用其迭代器来遍历集合，在每个元素上调用断言( predicate)，并利用迭代器的remove方法移除断言返回值为true的元素。其声明大致如下：

  ```java
  // Default method added to the Collection interface in Java 8
  default boolean removeIf(Predicate<? super E> filter) {
    Objects.requireNonNull(filter);
    boolean result = false;
    for (Iterator<E> it = iterator(); it.hasNext(); ) {
      if(filter.test(it.next())) {
        it.remove();
        result = true;
      }
    }
    return result;
  }
  ```

  ​	这是适用于removeIf方法的最佳通用实现，但遗憾的是，它在某些现实的Collection实现中会出错。比如，以`org.apache.commons.collections4.Collection.SynchronizedCollection`为例，这个类来自 Apache Commons类库，类似于java.util中的静态工厂Collections.synchronizedCollection。 Apache版本额外提供了利用客户端提供的对象(而不是用集合)进行锁定的功能。换句话说，它是一个包装类(E18)，它的所有方法在委托给包装集合之前，都在锁定对象上进行了同步。

  ​	**有了缺省方法，接口的现有实现就不会出现编译时没有报错或警告，运行时却失败的情况**。这个问题虽然并非普遍，但也不是孤立的意外事件。Java8在集合接口中添加的许多方法是极易受影响的，有些现有实现已知将会受到影响。

  ​	<u>建议尽量避免利用缺省方法在现有接口上添加新的方法，除非有特殊需要，但就算在那样的情况下也应该慎重考虑：缺省的方法实现是否会破坏现有的接口实现</u>。然而，在创建接口的时候，用缺省方法提供标准的方法实现是非常方便的，它简化了实现接口的任务(E20)。

  ​	**还要注意的是，缺省方法不支持从接口中删除方法，也不支持修改现有方法的签名。对接口进行这些修改肯定会破坏现有的客户端代码**。

---

+ 小结

​	结论很明显：**尽管缺省方法现在已经是Java平台的组成部分，但谨慎设计接口仍然是至关重要的**。虽然缺省方法可以在现有接口上添加方法，但这么做还是存在着很大的风险。就算接口中只有细微的缺陷都可能永远给用户带来不愉快；假如接口有严重的缺陷，则可能摧毁包含它的API。

## E22 接口只用于定义类型

+ 概述

  ​	当类实现接口时，接口就充当可以引用这个类的实例的类型(type)。因此，类实现了接口，就表明客户端可以对这个类的实例实施某些动作。**为了任何其他目的而定义接口是不恰当的**。

  ​	有一种接口被称为常量接口(constant interface)，它不满足上面的条件。这种接口不包含任何方法，它只包含静态的final域，每个域都导出一个常量。使用这些常量的类实现这个接口，以避免用类名来修饰常量名。下面举个例子：

  ```java
  // Constant interface antipattern - do not use!
  public interface PhysicalConstants {
    // Avogadro's number (1/mol)
    static final double AVOGADROS_NUMBER = 6.022_140_857e23;
    
    ...
  }
  ```

  ​	**常量接口模式是对接口的不良使用**。类在内部使用某些常量，这纯粹是实现细节。实现常量接口会导致把这样的实现细节泄露到该类的导出API中。类实现常量接口对于该类的用户而言并没有什么价值。实际上，这样做反而会使他们更加糊涂。更糟糕的是，它代表了一种承诺：如果在将来的发行版本中，这个类被修改了，它不再需要使用这些常量了，它依然必须实现这个接口，以确保二进制兼容性。如果非final类实现了常量接口，它的所有子类的命名空间也会被接口中的常量所"污染"。

  ​	**在Java平台类库中有几个常量接口，例如`java.io.ObjectStreamConstants`。这些接口应该被认为是反面的典型，不值得效仿**。

---

+ 导出常量的合理方案

  ​	如果要导出常量，可以有几种合理的选择方案。如果这些常量与某个现有的类或者接口紧密相关，就应该把这些常量添加到这个类或者接口中。

  ​	例如，在Java平台类库中所有的数值包装类，如Integer和Double，都导出了MIN_VALUE和MAX_VALUE常量。**如果这些常量最好被看作枚举类型的成员，就应该用枚举类型(enum type)（E34）来导出这些常量。否则，应该使用不可实例化的工具类(utility class)（E4）来导出这些常量**。下面的例子是前面的PhysicalConstants例子的工具类翻版：

  ```java
  // Constatnt utility class
  package com.effectivejava.science;
  
  public class PhysicalConstants {
    private PhysicalConstants() { } // Prevents instantiaion
    
    public static final double AVOGADROS_NUMBER = 6.022_140_857e23;
    
    ...
  }
  ```

  > 注意，有时候会在数字的字面量中使用下划线（_），Java7起支持，可提高可读性。
  >
  > 对于基数为10的字面量，无论是整数还是浮点数，都应该用下划线把数字隔成每三位一组，表示一千的正负倍数。

  ​	工具类通常要求客户端要用类名来修饰这些常量名，例如 PhysicalConstants.AVOGADROS_NUMBER。如果大量利用工具类导出的常量，可以通过利用静态导入（static import）机制，避免用类名来修饰常量名。

---

+ 小结

  ​	简而言之，**接口应该植被用来定义类型，不应该被用来导出常量**。

## E23 类层次优于标签类

+ 概述

  ​	**标签类过于冗长、容易出错，并且效率低下**。

  ​	幸运的是，面向对象的语言(如Java)提供了其他更好的方法来定义能表示多种风格对象的单个数据类型：子类型化(subtyping)。**标签类正是对类层次的一种简单的仿效**。

  ​	为了将标签类转变成类层次，首先要为标签类中的每个方法都定义一个包含抽象方法的抽象类，标签类的行为依赖于标签值。在 Figure类中，只有一个这样的方法：area。这个抽象类是类层次的根(root)。如果还有其他的方法其行为不依赖于标签的值，就把这样的方法放在这个类中。同样地，如果所有的方法都用到了某些数据域，就应该把它们放在这个类中。在 Figure类中，不存在这种类型独立的方法或者数据域。

  ​	接下来，为每种原始标签类都定义根类的具体子类。在前面的例子中，这样的类型有两个：圆形(Circle)和矩形(Rectangle)。在每个子类中都包含特定于该类型的数据域。在我们的示例中，radius是特定于圆形的, length和width是特定于矩形的。同时在每个子类中还包括针对根类中每个抽象方法的相应实现。以下是与原始的 Figure类相对应的类层次

  ```java
  // Class hierarchy replacement for a tagged class
  abstract class Figure {
    abstract double area();
  }
  
  class Circle extends Figure {
    final double radius;
    Circle(double radius) { this.radius = radius; }
    @Override double area() { return Math.PI * (radius*radius); }
  }
  
  class Rectangle extends Figure {
    final double length;
    final double width;
    Rectangle(double length, double width) {
      this.length = length;
      this.width = width;
    }
    @Override double area() { return length * width; }
  }
  ```

  ​	这个类层次纠正了前面提到过的标签类的所有缺点。这段代码简单且清楚，不包含在原来的版本中见到的所有样板代码。每个类型的实现都配有自己的类,这些类都没有受到不相关数据域的拖累。所有的域都是 final的。编译器确保每个类的构造器都初始化它的数据域，对于根类中声明的每个抽象方法都确保有一个实现。这样就杜绝了由于遗漏 switch case而导致运行时失败的可能性。

  ​	类层次的另一个好处在于，它们可以用来反映类型之间本质上的层次关系，有助于增强灵活性，并有助于更好地进行编译时类型检査。

---

+ 标签类举例

  ```java
  // Tagged class - vastly inferior to a class hierarchy!
  class Figure {
    enum Shape { RECTANGLE, CIRCLE };
    
    // Tag field - the shape of this figure
    final Shape shape;;
    
    // These fields are use only if shape is RECTANGLE
    double length;
    double width;
    
    // This field is used only if shape is CIRCLE
    double radius;
    
    // Constructor for circle
    Figure(couble radius) {
      shape = Shape.CIRCLE;
      this.radius = radius;
    }
    
    // Constructor for rectangle
    Figure(double length, double width) {
      shape = Shape.RECTANGLE;
      this.length = length;
      this.width = width;
    }
    
    double area() {
      switch(shape) {
        case RECTANGLE:
          return length * width;
        case CIRCLE:
          return Math.PI * (radius * radius);
        default:
          throw new AssertionError(shape);
      }
    }
  }
  ```

---

+ 小结

  ​	简而言之，标签类很少有适用的时候。当你想要编写一个包含显式标签域的类时，应该考虑一下，这个标签是否可以取消，这个类是否可以用类层次来代替。当你遇到一个包含标签域的现有类时，就要考虑将它重构到一个层次结构中去。

## * E24 静态成员类优于非静态成员类

+ 概述
  ​	嵌套类(nested class)是指定义在另一个类的内部的类。**嵌套类存在的目的应该只是为它的外围类(enclosing class)提供服务**。**如果嵌套类将来可能会用于其他的某个环境中，它就应该是顶层类( top-level class)**。

  ​	嵌套类有四种：

  + **静态成员类(static member class)**
  + **非静态成员类(nonstatic member class)**
  + **匿名类(anonymous class)**
  + **局部类(local class)**

  ​	除了第一种之外，其他三种都称为内部类(inner class）。本条目将告诉你什么时候应该使用哪种嵌套类，以及这样做的原因。

---

+ 静态成员类和非静态成员类

  ​	静态成员类是最简单的一种嵌套类。最好把它看作是普通的类，只是碰巧被声明在另个类的内部而已，它可以访问外围类的所有成员，包括那些声明为私有的成员。**静态成员类是外围类的一个静态成员，与其他的静态成员一样，也遵守同样的可访问性规则**。如果它被声明为私有的，它就只能在外围类的内部才可以被访问，等等。

  ​	<u>静态成员类的一种常见用法是作为公有的辅助类</u>，只有与它的外部类一起使用才有意义。例如，以枚举为例，它描述了计算器支持的各种操作(E34)。Operation枚举应该是Calculator类的公有静态成员类，之后Calculator类的客户端就可以用诸如Calculator.Operation.PLUS和Calculator.Operation.MINUS这样的名称来引用这些操作。

  ​	从语法上讲，静态成员类和非静态成员类之间唯一的区别是，静态成员类的声明中包含修饰符static。尽管它们的语法非常相似，但是这两种嵌套类有很大的不同。非静态成员类的每个实例都隐含地与外围类的一个外围实例(enclosing instance)相关联。在非静态成员类的实例方法内部，可以调用外围实例上的方法，或者利用修饰过的this(qualified this)构造获得外围实例的引用[JLS,15.8.4]。**如果嵌套类的实例可以在它外围类的实例之外独立存在，这个嵌套类就必须是静态成员类：在没有外围实例的情况下，要想创建非静态成员类的实例是不可能的。**

  ​	<u>当非静态成员类的实例被创建的时候，它和外围实例之间的关联关系也随之被建立起来；而且，这种关联关系以后不能被修改</u>。通常情况下，当在外围类的某个实例方法的内部调用非静态成员类的构造器时，这种关联关系被自动建立起来。使用表达式`enclosingInstance. new MemberClass(args)`来手工建立这种关联关系也是有可能的，但是很少使用。正如你所预料的那样，这种关联关系需要消耗非静态成员类实例的空间，并且会增加构造的时间开销。

  ​	**非静态成员类的一种常见用法是定义一个 Adapter，它允许外部类的实例被看作是另一个不相关的类的实例**。例如，Map接口的实现往往使用非静态成员类来实现它们的集合视图(collection view)，这些集合视图是由Map的 keySet、 entrySet和values方法返回的。同样地，诸如Set和List这种集合接口的实现往往也使用非静态成员类来实现它们的迭代器(Iterator)：

  ```java
  // Typical use of a nonstatic member class
  public class MySet<E> extends AbstractSet<E> {
    ... // Bulk of the class omitted
      
    @Override public Iterator<E> iterator() {
      return new MyIterator();
    }
    
    private class MyIterator implements Iterator<E> {
      ...
    }
  }
  ```

  ​	**如果声明成员类不要求访问外围实例，就要始终把修饰符static放在它的声明中，使它成为静态成员类，而不是非静态成员类**。如果省略了static修饰符，则每个实例都将包含一个额外的指向外围对象的引用。如前所述，<u>保存这份引用要消耗时间和空间，并且会导致外围实例在符合垃圾回收（E7）时却仍然得以保留。由此造成的内存泄漏可能是灾难性的</u>。但是常常难以发现，因为这个引用是不可见的。

  ​	私有静态成员类的一种常见用法是代表外围类所代表的对象的组件。以Map实例为例，它把键(key)和值( value)关联起来。许多Map实现的内部都有一个Entry对象，对应于Map中的每个键-值对。虽然每个entry都与一个Map关联，但是 <u>entry上的方法(getKey、 getValue和 setValue)并不需要访问该Map。因此，使用非静态成员类来表示entry是很浪费的：私有的静态成员类是最佳的选择</u>。如果不小心漏掉了entry声明中的 static修饰符，该Map仍然可以工作，但是每个 entry中将会包含一个指向该Map的引用，这样就浪费了空间和时间。

  ​	如果相关的类是导出类的公有或受保护的成员，毫无疑问，在静态和非静态成员类之间做出正确的选择是非常重要的。在这种情况下，该成员类就是导出的API元素，在后续的发行版本中，如果不违反向后兼容性，就无法从非静态成员类变为静态成员类。

---

+ 匿名类

  ​	顾名思义，匿名类是没有名字的。**它不是外围类的一个成员**。它并不与其他的成员起被声明，而是在使用的同时被声明和实例化。<u>匿名类可以出现在代码中任何允许存在表达式的地方</u>。**当且仅当匿名类出现在非静态的环境中时，它才有外围实例**。**但是即使它们出现在静态的环境中，也不可能拥有任何静态成员，而是拥有常数变量(constant variable)，常数变量是final基本类型，或者被初始化成常量表达式[JLS, 4.12.4]的字符串域**。

  ​	<u>匿名类的运用受到诸多的限制。除了在它们被声明的时候之外，是无法将它们实例化的。不能执行 instanceof测试，或者做任何需要命名类的其他事情</u>。无法声明一个匿名类来实现多个接口，或者扩展一个类，并同时扩展类和实现接口。**除了从超类型中继承得到之外，匿名类的客户端无法调用任何成员**。由于匿名类出现在表达式中，它们必须保持简短大约10行或者更少)，否则会影响程序的可读性。

  ​	**在Java中增加lambda（E6）之前，匿名类是动态地创建小型函数对象(function object)和过程对象( process object)的最佳方式，但是现在会优先选择lambda详见E42)。匿名类的另一种常见用法是在静态工厂方法的内部(参见E20中的 intArrayAsList方法)**

---

+ 局部类

  ​	局部类是四种嵌套类中使用最少的类。<u>在任何"可以声明局部变量"的地方，都可以声明局部类，并且局部类也遵守同样的作用域规则</u>。局部类与其他三种嵌套类中的每一种都有一些共同的属性。与成员类一样，局部类有名字，可以被重复使用。**与匿名类一样，只有当局部类是在非静态环境中定义的时候，才有外围实例，它们也不能包含静态成员**。与匿名类一样，它们必须非常简短，以便不会影响可读性。

---

+ 小结

  ​	总而言之，共有四种不同的嵌套类，每一种都有自己的用途。

  + **如果一个嵌套类需要在单个方法之外仍然是可见的，或者它太长了，不适合放在方法内部，就应该使用成员类。**

  + **如果成员类的每个实例都需要一个指向其外围实例的引用，就要把成员类做成非静态的；否则，就做成静态的。**

  + **假设这个嵌套类属于一个方法的内部，如果你只需要在一个地方创建实例，并且已经有了一个预置的类型可以说明这个类的特征，就要把它做成匿名类；否则，就做成局部类。**

## E25 限制源文件为单个顶级类

+ 概述

​	虽然Java编译器允许在一个源文件中定义多个顶级类，但这么做并没有什么好处，只会带来巨大的风险。因为**在一个源文件中定义多个顶级类，可能导致给一个类提供多个定义**。<u>哪一个定义会被用到，取决于源文件被传给编译器的顺序</u>。

​	为了具体地说明，下面举个例子，这个源文件中只包含一个Main类，它将引用另外两个顶级类（Utensil和Dessert）的成员：

```java
public class Main {
  public static void main(String[] args) {
    System.out.println((Utensil.NAME + Dessert.NAME));
  }
}
```

​	现在假设在一个名为Utensil.java的源文件中同时定义了Utensil和Dessert：

```java
// Two classes defined in one file. Don't ever do this!
class Utensil {
  static final String NAME = "pan";
}

class Dessert {
 	static final String NAME = "cake";
}
```

​	当然，主程序会打印出"pancacke"。

​	现在假设不小心在另一个名为Dessert.java的源文件中也定义了同样的两个类：

```java
// Two classes defined in one file. Don't ever do this!
class Utensil {
  static final String NAME = "pot";
}

class Dessert {
 	static final String NAME = "pie";
}
```

​	如果你侥幸是用命令`javac Main. Java Dessert.java`来编译程序，那么编译就会失败，此时编译器会提醒你定义了多个Utensil和Dessert类。这是因为编译器会先编译`Main.java`，当它看到Utensil的引用(在Dessert引用之前)，就会在`Utensil.java`中查看这个类，结果找到Utensil和 Dessert这两个类。当编译器在命令行遇到 `Dessert.java`时，也会去查找该文件，结果会遇到Utensil和 Dessert这两个定义。

​	如果用命令`javac Main.java`或者`javac Main. java Utensil.java`编译程序，结果将如同你还没有编写 `Dessert.java`文件一样，输出pancake。但如果是用命令`javac Dessert.java Main.java`编译程序，就会输出 potpie。**程序的行为受源文件被传给编译器的顺序影响，这显然是让人无法接受的**。

---

+ 问题解决

​	这个问题的修正方法很简单，只要把顶级类(本例中是指Utensil和Dessert)分别放入独立的源文件即可。**如果一定要把多个顶级类放进一个源文件中，就要考虑使用静态成员类（E24），以此代替将这两个类分到独立源文件中去**。

​	如果这些类服从于另个类，那么将它们做成静态成员类通常比较好，因为这样增强了代码的可读性，如果将这些类声明为私有的（E15），还可以使它们减少被读取的概率。以下就是做成静态成员类的范例：

```java
// Static member classes instead of multiple top-level classes
public class Test {
  public static void main(String[] args) {
    System.out.println(Utensil.NAME + Dessert.NAME);
  }
  
  private static class Utensil {
		static final String NAME = "pan";
  }
  
  private static class Dessert {
		static final String NAME = "cake";
  }
}
```

---

+ 小结

​	结论显而易见：**永远不要把多个顶级类或者接口放在一个源文件中**。

​	遵循这个规则可以确保编译时一个类不会有多个定义。这么做反过来也能确保编译产生的类文件，以及程序结果的行为，都不会受到源文件被传给编译器时的顺序的影响。

# 5、泛型

> 从Java5开始，泛型(generic)已经成了Java编程语言的一部分。
>
> **在没有泛型之前，从集合中读取到的每一个对象都必须进行转换**。如果有人不小心插入了类型错误的对象，在运行时的转换处理就会出错。<u>有了泛型之后，你可以告诉编译器每个集合中接受哪些对象类型。编译器自动为你的插入进行转换，并在编译时告知是否插入了类型错误的对象</u>。这样可以使程序更加安全，也更加清楚，但是要享有这些优势(不限于集合)有一定的难度。
>
> 本章就是教你如何最大限度地享有这些优势，又能使整个过程尽可能简单化。

## * E26 请不要使用原生态类型

+ 概述

  ​	先介绍一些术语。**声明中具有一个或者多个类型参数(type parameter)的类或者接口，就是泛型(generic)类或者接口**[JLS，8.1.2，9.1.2]。

  ​	例如，List接口就只有单个类型参数E，表示列表的元素类型。这个接口的全称是List\<E\>(读作"E的列表")，但是人们经常把它简称为List。**泛型类和接口统称为泛型(generic type)**。

  ​	每一种泛型定义一组参数化的类型(parameterized type)，构成格式为：先是类或者接口的名称，接着用尖括号(\<\>)把对应于泛型形式类型参数的实际类型参数(actual typeparamter)列表[JLS，4.4，4.5]括起来。例如，List\<String\>(读作"字符串列表")是个参数化的类型，表示元素类型为String的列表。(String是与形式的类型参数E相对应的实际类型参数。)

  ​	最后一点，**每一种泛型都定义一个原生态类型(raw type)，即不带任何实际类型参数的泛型名称**[JLS，4.8]。例如，与List\<E\>相对应的原生态类型是List。**原生态类型就像从类型声明中删除了所有泛型信息一样。它们的存在主要是为了与泛型出现之前的代码相兼容**。

  ​	在Java增加泛型之前，下面这个集合声明是值得参考的。从Java9开始，它依然合法，但是已经没什么参考价值了：

  ```java
  // Raw collection type = don't do this!
  // My stamp collection. Contains only Stamp instances;
  private final Collection stamps = ... ;
  ```

  ​	如果现在使用这条声明，并且不小心将一个coin放进了stamp集合中，这一错误的插入照样得以编译和运行，不会出错（不过编译器确实会发出一条模糊的警告信息）：

  ```java
  // Erroneous insertion of coin into stamp collection
  stamps.add(new Coin(...)); // Emits "unchecked call" warning
  ```

  ​	直到从stamp集合中获取coin时才会收到一条错误提示：

  ```java
  // Raw iterator type - don't do this!
  for(Iterator i = stamps.iterator(); i.hasNext();)
    Stamp stamp = (Stamp) i.next(); // Throws ClassCastException
  	stamp.cancel();
  ```

  ​	如本书中经常提到的，**出错之后应该尽快发现，最好是编译时就发现**。在本例中，直到运行时才发现错误，已经出错很久了，而且它在代码中所处的位置，距离包含错误的这部分代码已经很远了。一旦发现 ClassCastException，就必须搜索代码，查找将coin放进stamp集合的方法调用。此时编译器帮不上忙，因为它无法理解这种注释"Contains only Stamp instances"(只包含 Stamp实例)。

  ​	有了泛型之后，类型声明中可以包含以下信息，而不是注释：

  ```java
  // Parameterized collection type - typesafe
  private final Collection<Stamp> stamps = ... ;
  ```

  ​	通过这条声明，编译器知道 stamps应该只包含 Stamp实例，并给予保证( guarantee)假设整个代码库在编译过程中都没有发出(或者隐瞒，E27)任何警告。当 stamps利用一个参数化的类型进行声明时，错误的插入会产生一条编译时的错误消息，告诉你具体是哪里出错了：

  ```shell
  Test.java:9:error:incompatible types: Coin cannot be converted to Stamp
  	c.add(new Coin());
  						^
  ```

  ​	**<u>从集合中检索元素时，编译器会替你插入隐式的转换，并确保它们不会失败(依然假设所有代码都没有产生或者隐瞒任何编译警告)</u>**。假设不小心将coin插入 stamp集合，这显得有点牵强，但这类问题却是真实的。例如，很容易想象有人会不小心将一个BigInteger实例放进一个原本只包含 BigDecimal实例的集合中。

  ​	如上所述，使用原生态类型(没有类型参数的泛型)是合法的，但是永远不应该这么做。**如果使用原生态类型，就失掉了泛型在安全性和描述性方面的所有优势**。

  ​	**既然不应该使用原生态类型，为什么Java语言的设计者还要允许使用它们呢？这是为了提供兼容性**。

  ​	因为泛型出现的时候，Java平台即将进入它的第二个十年，已经存在大量没有使用泛型的Java代码。人们认为让所有这些代码保持合法，并且能够与使用泛型的新代码互用，这点很重要。它必须合法才能将参数化类型的实例传递给那些被设计成使用普通类型的方法，反之亦然。<u>这种需求被称作**移植兼容性(Migration Compatibility)**，促成了支持原生态类型，以及利用**擦除(erasure)**(E28)实现泛型的决定</u>。

  ​	虽然不应该在新代码中使用像List这样的原生态类型，使用参数化的类型以允许插入任意对象(比如List\< Object\>)是可行的。原生态类型List和参数化的类型List\<Object\>之间到底有什么区别呢？不严格地说，<u>前者逃避了泛型检査，后者则明确告知编译器，它能够持有任意类型的对象</u>。

  ​	虽然可以将List\<String\>传递给类型List的参数，但是不能将它传给类型List\<Object\>的参数。**<u>泛型有子类型化(subtyping)的规则，List\<String\>是原生态类型List的一个子类型，而不是参数化类型List\<Object\>的子类型(E28)</u>**。因此，**如果使用像List这样的原生态类型，就会失掉类型安全性，但是如果使用像List\<Object\>这样的参数化类型，则不会**。

  ​	为了更具体地说明，请参考以下程序：

  ```java
  // Fails at runtime - unsafeAdd method uses a raw type (List)!
  public static void main(String[] args) {
    List<String> strings = new ArrayList<>();
    unsafeAdd(strings, Integer.valueOf(42));
    String s = strings.get(0); // Has compiler-generated case
  }
  
  private static void unsafeAdd(List list, Object o){
    list.add(o);
  }
  ```

  ​	这段程序可以进行编译，但是因为它使用了原生态类型List，你会收到一个警告：

  ```shell
  Test.java:10:warning: [unchecked] unchecked call to add(E) as a member of the raw type list
  	list.add(o);
  					 ^
  ```

  ​	实际上，如果运行这段程序，在程序试图将`strings.get(0)`的调用结果Integer转换成String时,你会收到一个ClassCastException异常。<u>这是一个编译器生成的转换，因此一般保证会成功，但是我们在这个例子中忽略了一条编译器警告，为此付出了代价</u>。

  ​	如果在 unsafeAdd声明中用参数化类型工List\<Object\>代替原生态类型List，并试着重新编译这段程序，会发现它无法再进行编译了，并发出以下错误消息

  ```shell
  Test.java:5: error: incompatible types: List<String> cannot be converted to List<Object>
  	unsafeAdd(strings, Integer.valueOf(42));
  			^
  ```

  ​	在不确定或者不在乎集合中的元素类型的情况下，你也许会使用原生态类型。例如，假设想要编写一个方法，它有两个集合，并从中返回它们共有元素的数量。如果你对泛型还不熟悉，可以参考以下方式来编写这种方法：

  ```java
  // Use of raw type for unknown element type - don't do this!
  static int numElementsInCommon(Set s1, Set s2) {
    int result = 0;
    for(Object o1 : s1)
      if(s2.contains(o1))
        result++;
    return result;
  }
  ```

  ​	这个方法可行，但它使用了原生态类型，这是很危险的。安全的替代做法是<u>使用无限制的通配符类型</u>( unbounded wildcard type)。**如果要使用泛型，但不确定或者不关心实际的类型参数，就可以用一个问号代替**。例如，泛型Set\<E\>的无限制通配符类型为Set\<?\>(读作"某个类型的集合")。这是最普通的参数化Set类型，可以持有任何集合。下面是numElementsInCommon方法使用了无限制通配符类型时的情形：

  ```java
  // Uses unbounded wildcard type - typesafe and flexible
  static int numElementsInCommon(Set<?> s1, Set<?> s2) { ... }
  ```

  ​	无限制通配类型Set\<?\>和原生态类型Set之间有什么区别呢？这个问号真正起到作用了吗？这一点不需要赘述，**但通配符类型是安全的，原生态类型则不安全**。由于可以将任何元素放进使用原生态类型的集合中，因此很容易破坏该集合的类型约束条件(如之前范例中所示的 unsafeAdd方法)；但**不能将任何元素(除了null之外)放到collection\<?\>中**。如果尝试这么做，将会产生一条像这样的编译时错误消息：

  ```shell
  WildCard.java:13: error: incompatible types: String cannot be converted to CAP#1
    c.add("verboten");
  				^
  	where CAP#1 is a fresh type-variable
  	 CAP#1 extends Object from capture of ?
  ```

  ​	这样的错误消息显然无法令人满意，但是编译器已经尽到了它的职责，防止你破坏集合的类型约束条件。你不仅无法将任何元素(除了null之外)放进`Collection<?>`中而且根本无法猜测你会得到哪种类型的对象。要是无法接受这些限制，就可以使用**泛型方法(E30)**或者**有限制的通配符类型(E31)**。

  ​	不要使用原生态类型，这条规则有几个小小的例外。**必须在类文字(class literal)中使用原生态类型**。规范不允许使用参数化类型(虽然允许数组类型和基本类型[JLS, 15.8.2]换句话说，**<u>List.class、String[].class和int.class都合法，但是List\<String\>.class和List\<?\>. class则不合法</u>**。

  ​	这条规则的第二个例外与instanceof操作符有关。**<u>由于泛型信息可以在运行时被擦除，因此在参数化类型而非无限制通配符类型上使用instanceof操作符是非法的</u>**。

  ​	**<u>用无限制通配符类型代替原生态类型，对 instanceof操作符的行为不会产生任何影响</u>**。在这种情况下，尖括号(\<\>)和问号(?)就显得多余了。

  ​	**下面是利用泛型来使用instanceof操作符的首选方法**：

  ```java
  // Legitimate use of raw type - instanceof operator
  if(o instanceof Set) {		// Raw type
    Set<?> s = (Set<?>) o;	// Wildcard type
    ...
  }
  ```

  ​	<u>注意，一旦确定这个o是个Set，就必须将它转换成通配符类型Set\<?>，而不是转换成原生态类型Set。这是个受检的(checked)转换，因此不会导致编译时警告</u>。

---

+ 小结

  总而言之，**使用原生态类型会在运行时导致异常，因此不要使用**。

  **原生态类型只是为了与引入泛型之前的遗留代码进行兼容和互用而提供的**。

  让我们做个快速的回顾：

  + Set\<Object\>是个<u>参数化类型</u>，表示可以包含<u>**任何**对象类型</u>的一个集合；
  + Set\<?\>则是个<u>通配符类型</u>，表示只能包含<u>**某种**未知对象类型</u>的一个集合；
  + Set是一个<u>原生态类型</u>，它脱离了泛型系统。

  **前两种是安全的，最后一种不安全**。

​	为便于参考，在下表中概括了本条目中所介绍的术语(及本章后续条目中将要介绍的些术语)：

| 术语             | 范例                                  | 条目     |
| ---------------- | ------------------------------------- | -------- |
| 参数化的类型     | List\<String\>                        | E26      |
| 实际类型参数     | String                                | E26      |
| 泛型             | List\<E\>                             | E26、E29 |
| 形式类型参数     | E                                     | E26      |
| 无限制通配符类型 | List\<?\>                             | E26      |
| 原生态类型       | List                                  | E26      |
| 有限制类型参数   | \<E extends Number\>                  | E29      |
| 递归类型限制     | \<T extends Comparable\<T\>\>         | E30      |
| 有限制通配符类型 | List\<? Extends Number\>              | E31      |
| 泛型方法         | static\<E\> List\<E\> asList(E\[\] a) | E30      |
| 类型令牌         | String.class                          | E33      |

## * E27 消除非受检的警告

+ 概述

  ​	用泛型编程时会遇到许多编译器警告：非受检转换警告(unchecked cast warning)、非受检方法调用警告、非受检参数化可变参数类型警告(unchecked parameterized vararg type warning)，以及非受检转换警告(unchecked conversion warning)。当你越来越熟悉泛型之后遇到的警告也会越来越少，但是<u>不要期待一开始用泛型编写代码就可以正确地进行编译</u>。

  ​	有许多非受检警告很容易消除。例如，假设意外地编写了这样一个声明：

  ```java
  Set<Lark> exalation = new HashSet();
  ```

  ​	编译器会细致地提醒你哪里出错了：

  ```shell
  Venery.java:4: warning: [unchecked] unchecked conversion Set<Lark> exaltation = new HashSet();
  
  require: Set<Lark>
  found: HashSet
  ```

  ​	你就可以纠正所显示的错误，消除警告。<u>注意，不必真正去指定类型参数，只需要用在Java7中开始引入的菱形操作符(diamond operator)(\<\>)将它括起来即可。随后编译器就会推测出正确的实际类型参数</u>(在本例中是Lark)：

  ```java
  Set<Lark> exaltation= new Hashset<>();
  ```

  ​	有些警告非常难以消除。本章主要介绍这种警告示例。当你遇到需要进行一番思考的警告时，要坚持住！**要尽可能地消除每一个非受检警告**。如果消除了所有警告，就可以确保代码是类型安全的，这是一件很好的事情。这意味着不会在运行时出现Class-Cast-Exception异常，你会更加自信自己的程序可以实现预期的功能。

  ​	**如果无法消除警告，同时可以证明引起警告的代码是类型安全的，(只有在这种情况下才可以用一个 @SuppressWarnings("unchecked")注解来<u>禁止</u>这条警告**。如果在禁止警告之前没有先证实代码是类型安全的，那就只是给你自己一种错误的安全感而已。代码在编译的时候可能没有出现任何警告，但它在运行时仍然会抛出ClassCastException异常。但<u>是如果忽略(而不是禁止)明知道是安全的非受检警告，那么当新出现一条真正有问题的警告时，你也不会注意到。新出现的警告就会淹没在所有的错误警告声当中</u>。

  ​	<u>@SuppressWarnings注解可以用在任何粒度的级别中，从单独的局部变量声明到整个类都可以</u>。**<u>应该始终在尽可能小的范围内使用@SuppressWarnings注解</u>**。它通常是个变量声明，或是非常简短的方法或构造器。**永远不要在整个类上使用@SuppressWarnings，这么做可能会掩盖重要的警告**。

  ​	<u>如果你发现自己在长度不止一行的方法或者构造器中使用了@SuppressWarnings注解，可以将它移到一个局部变量的声明中。虽然你必须声明一个新的局部变量，不过这么做还是值得的</u>。例如，看看ArrayList类当中的toArray方法：

  ```java
  public <T> T[] toArray(T[] a) {
    if(a.length < size)
      return (T[]) Arrays.copyOf(elements, size, a.getClass());
    System.arraycopy(elements, 0, a, 0, size);
    if(a.length > size)
      a[size] = null;
    return a;
  }
  ```

  ​	如果编译ArrayList，该方法就会产生这条警告：

  ```shell
  ArrayList.java:305: warning: [unchecked] unchecked cast return (T[]) Arrays.copyOf(elements, size, a.getClass());
  							^
  	required: T[]
  	found:		Object[]
  ```

  ​	将 SuppressWarnings注解放在 return语句中是合法的，因为它不是声明[JLS，9.7]。<u>你可以试着将注解放在整个方法上，但是在实践中千万不要这么做，而是应该声明个局部变量来保存返回值，并注解其声明</u>，像这样：

  ```java
  // Adding local variable to reduce scope of @SuppressWarnings
  public <T> T[] toArray(T[] a) {
    if (a.length < size) {
      // This cast is correct because the array we're creating
      // is of the same type as the one passed in, which is T[].
      @SuppressWarnings("unchecked") T[] result = (T[]) Arrays.copyOf(elements, size, a.getClass());
      return result;
    }
    System.arraycopy(elements, 0, a, 0, size);
    if(a.length > size)
      a[size] = null;
    return a;
  }
  ```

  ​	这个方法可以正确地编译，禁止非受检警告的范围也会减到最小。

  ​	**每当使用@Suppresswarnings("unchecked")注解时，都要添加一条注释，说明为什么这么做是安全的**。这样可以帮助其他人理解代码，更重要的是，可以尽量减少其他人修改代码后导致计算不安全的概率。如果你觉得这种注释很难编写，就要多加思考。最终你会发现非受检操作是非常不安全的。

---

+ 小结

​	总而言之，非受检警告很重要，不要忽略它们。每一条警告都表示可能在运行时抛出ClassCastException异常。要尽最大的努力消除这些警告。

​	如果无法消除非受检警告，同时可以证明引起警告的代码是类型安全的，就可以在尽可能小的范围内使用`@SuppressWarnings("unchecked″)`注解禁止该警告。要用注释把禁止该警告的原因记录下来。

## * E28 List优于数组

+ 概述

  ​	数组与泛型相比，有两个重要的不同点。
  + 首先，**数组是协变的(covariant)**。这个词听起来有点吓人，其实只是表示如果Sub为Super的子类型，那么数组类型Sub\[\]就是Super\[\]的子类型。
  + 相反，**泛型则是不可变的(invariant)**：对于任意两个不同的类型Type1和Type2，List\<Type1\>既不是List\<Type2\>的子类型，也不是List\<Type2\>的超类型[JLS, 4.10; Naftalin07, 2.5]。

  ​	你可能认为，这意味着泛型是有缺陷的，但**实际上可以说数组才是有缺陷的**。

  下面的代码片段是合法的：

  ```java
  // Fails at runtime!
  Object[] objectArray = new Long[1];
  objectArray[0] = "I don't fit in"; // Throws ArrayStoreException
  ```

  但下面这段代码则不合法：

  ```java
  // Won't compile!
  List<Object> ol = new ArrayList<Long>(); // Incompatible types
  ol.add("I don't fit in");
  ```

  ​	这其中无论哪一种方法，都不能将String放进Long容器中，但是利用数组，你会在运行时才发现所犯的错误；而利用列表，则可以在编译时就发现错误。我们当然希望在编译时就发现错误。

  ​	数组与泛型之间的第二大区别在于，数组是具体化的(reified)[JLs，4.7]。因此数组会在运行时知道和强化它们的元素类型。如上所述，如果企图将String保存到Long数组中，就会得到一个 ArrayStoreException异常。相比之下，泛型则是通过擦除(erasure)[JLS，4.6]来实现的。这意味着，**泛型只在编译时强化它们的类型信息，并在运行时丢弃(或者擦除)它们的元素类型信息。擦除就是使泛型可以与没有使用泛型的代码随意进行互用(E26)，以确保在Java5中平滑过渡到泛型**。

  ​	由于上述这些根本的区别，因此**数组和泛型不能很好地混合使用**。

  ​	例如，**创建泛型参数化类型或者类型参数的数组是非法的**。这些数组创建表达式没有一个是合法的：`new List<E>[]`、 `new List<String>[]`和`new E[]`。这些在编译时都会导致一个泛型数组创建(generic array creation)错误。

  ​	**为什么创建泛型数组是非法的？因为它不是类型安全的。要是它合法，编译器在其他正确的程序中发生的转换就会在运行时失败，并出现一个ClassCastException异常。这就违背了泛型系统提供的基本保证**。

  ​	为了更具体地对此进行说明，以下面的代码片段为例：

  ```java
  // Why generic array creation is illegal - won't compile!
  List<String>[] stringLists = new List<String>[1];		// (1)
  List<Integer> intList = List.of(42);								// (2)
  Object[] objects = stringLists;											// (3)
  objects[0] = intList;																// (4)
  String s = stringLists[0].get(0);										// (5)
  ```

  ​	我们假设第1行是合法的，它创建了一个泛型数组。第2行创建并初始化了一个包含单个元素的List\< Integer\>。第3行将List\<String\>数组保存到一个Object数组变量中，这是合法的，因为数组是协变的。第4行将List\<Integer\>保存到 Object数组里唯一的元素中，这是可以的，因为**泛型是通过擦除实现的：List\< Integer\>实例的运行时类型只是List，List\<String\>[]实例的运行时类型则是List\[\]**，因此这种安排不会产生 ArrayStoreException异常。但现在我们有麻烦了。我们将一个List\<Integer\>实例保存到了原本声明只包含List\<String\>实例的数组中。在第5行中，我们从这个数组里唯一的列表中获取了唯一的元素。<u>编译器自动地将获取到的元素转换成String，但它是一个Integer，因此，我们在运行时得到了一个ClassCastException异常</u>。为了防止出现这种情况，(创建泛型数组的)第1行必须产生一条编译时错误。

  ​	**从技术的角度来说，像`E`、`List<E>`和`List<String>`这样的类型应称作不可具体化的(nonreifiable)类型**[JLS，4.7]。直观地说，<u>不可具体化的(non-reifiable)类型是指其运行时表示包含的信息比它的编译时表示包含的信息更少的类型</u>。

  ​	**唯一可具体化的(reifiable)参数化类型是无限制的通配符类型，如`List<?>`和`Map<?，?>`(E26)。虽然不常用，但是创建无限制通配类型的数组是合法的**。

  ​	**禁止创建泛型数组**可能有点讨厌。例如，这表明<u>泛型一般不可能返回它的元素类型数组</u>(部分解决方案请见E33)。这也意味着在结合使用可变参数(varargs)方法(E53)和泛型时会出现令人费解的警告。这是由于**每当调用可变参数方法时，就会创建个数组来存放 varargs参数**。如果这个数组的元素类型不是可具体化的( reifialbe)，就会得到一条警告。利用 SafeVarargs注解可以解决这个问题(详见第32条)。

  ​	**当你得到泛型数组创建错误时，最好的解决办法通常是优先使用集合类型`List<E>`，而不是数组类型`E[]`**。<u>这样可能会损失一些性能或者简洁性，但是换回的却是更高的类型安全性和互用性</u>。

  ​	例如，假设要通过构造器编写一个带有集合的Chooser类和一个方法，并用该方法返回在集合中随机选择的一个元素。根据传给构造器的集合类型，可以用chooser充当游戏用的色子、魔术8球(一种卡片棋牌类游戏)，或者一个蒙特卡罗模拟的数据源。

  ​	下面是一个没有使用泛型的简单实现：

  ```java
  // Chooser - a class badly in need of generics!
  public class Chooser {
    private final Object[] choiceArray;
    
    public Chooser(Collection choices) {
      choiceArray = choices.toArray();
    }
    
    public Object choose() {
      Random rnd = ThreadLocalRandom.current();
      return choiceArray[rnd.nextInt(choiceArray.length)];
    }
  }
  ```

  ​	<u>要使用这个类，必须将 choose方法的返回值，从Object转换成每次调用该方法时想要的类型，如果搞错类型，转换就会在运行时失败</u>。牢记（E29）的建议，努力将Chooser修改成泛型，修改部分如粗体所示：

  ```java
  // A first cut at making Chooser generic - won't compile
  public class Chooser<T> {
    private final T[] choiceArray;
    
    public Chooser(Collection<T> choices) {
      choiceArray = choices.toArray();
    }
    
    // choose method unchanged 
  }
  ```

  ​	如果尝试编译这个类，将会得到以下错误信息：

  ```shell
  Chooser.java:9: error: incompatible types: Object[] cannot be convert to T[]
  	choiceArray = choices.toArray();
  															^
  	where T is a type-variable:
  		T extends Object declared in class Chooser 
  ```

  ​	你可能会说：这没什么大不了的，我可以把Object数组转换成T数组：

  ```java
  choiceArray = (T[]) choices.toArray();
  ```

  ​	这样做的确消除了错误信息，但是现在得到了一条警告：

  ```shell
  Chooser.java:9: warning: [unchecked] unchecked cast
  	choiceArray = (T[]) choices.toArray();
  	required: T[], found: Object[]
  	where T is a type-variable:
  		T extends Object declared in class Chooser
  ```

  ​	**编译器告诉你，它无法在运行时检查转换的安全性，因为程序在运行时还不知道T是什么——记住，元素类型信息会在运行时从泛型中被檫除**。这段程序可以运行吗？可以，但是编译器无法证明这一点。你可以亲自证明，只要将证据放在注释中，用一条注解禁止警告，但是最好能消除造成警告的根源（E27）。

  ​	要消除未受检的转换警告，必须选择用List代替数组。下面是编译时没有出错或者警告的Chooser类版本：

  ```java
  // List-based Chooser - typesafe
  public class Chooser<T> {
    private final List<T> choiceList;
    
    public Chooser(Collection<T> choices) {
      choiceList = new ArrayList<>(choices);
    }
    
    public T choose() {
  		Random rnd = ThreadLocalRandom.current();
      return choiceList.get(rnd.nextInt(choiceList.size()));
    }
  }
  ```

  ​	这个版本的代码稍微冗长一点，运行速度可能也会慢一点，但是在运行时不会得到ClassCastException异常，为此也值了。

---

+ 小结

  ​	总而言之，数组和泛型有着截然不同的类型规则。

  + **数组是协变且可以具体化的**；
  + **泛型是不可变的且可以被擦除的**。

  ​	因此，数组提供了运行时的类型安全，但是没有编译时的类型安全，反之，对于泛型也一样。

  ​	**一般来说，数组和泛型不能很好地混合使用**。

  ​	<u>**如果你发现自己将它们混合起来使用，并且得到了编译时错误或者警告，你的第一反应就应该是用List代替数组**</u>。

## * E29 优先考虑泛型

+ 概述

  ​	一般来说，将集合声明参数化，以及使用JDK所提供的泛型方法，这些都不太困难。编写自己的泛型会比较困难一些，但是值得花些时间去学习如何编写。

  ​	E7中简单的（玩具）堆栈实现原本如下：

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

  ​	这个类应该先被参数化，但是它没有，我们可以在后面将它**泛型化(generify)**。换句话说，可以将它参数化，而又不破坏原来非参数化版本的客户端代码。也就是说，客户端必须转换从堆栈里弹出的对象，以及可能在运行时失败的那些转换。

  ​	<u>将类泛型化的第一步是在它的声明中添加一个或者多个类型参数</u>。

  ​	在这个例子中有一个类型参数，它表示堆栈的元素类型，这个参数的名称通常为E（E68）。

  ​	下一步是用相应的类型参数替换所有的Object类型，然后试着编译最终的程序：

  ```java
  // Initial attempt to generify Stack - won't compile!
  public class Stack<E> {
    private E[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;
  
    public Stack() {
      elements = new E[DEFAULT_INITIAL_CAPACITY];
    }
  
    public void push(E e) {
      ensureCapacity();
      elements[size++] = e;
    } 
  
    public E pop() {
      if(size == 0)
        throw new EmptyStackException();
      E result = elements[--size];
      elements[size] = null; // Eliminate obsolete reference
      return result;
    }
    ...// no changes in isEmpty or ensureCapacity
  }
  ```

  ​	通常，你将至少得到一个错误或警告，这个类也不例外。幸运的是，这个类只产生一个错误，内容如下：

  ```shell
  Stack.java:8: generic array creation
  	elements = new E[DEFAULT_INITIAL_CAPACITY];
  							^
  ```

  ​	如（E28）所述，**你不能创建不可具体化的(non-reifiable)类型的数组**，如E。每当编写用数组支持的泛型时，都会出现这个问题。解决这个问题有两种方法。<u>**第一种，直接绕过创建泛型数组的禁令：创建一个 Object的数组，并将它转换成泛型数组类型**。现在错误是消除了，但是编译器会产生一条警告。这种用法是合法的，但(整体上而言)不是类型安全的</u>：

  ```shell
  Stack.java:8: waring: [ubchecked] unchecked cast
  found: Object[], required: E[]
  	elements = (E[]) new Object[DEFAULT_INITIAL_CAPACITY];
  								^
  ```

  ​	编译器不可能证明你的程序是类型安全的，但是你可以。<u>你自己必须确保未受检的转换不会危及程序的类型安全性</u>。相关的数组(即elements变量)保存在一个私有的域中，永远不会被返回到客户端，或者传给任何其他方法。<u>这个数组中保存的唯一元素，是传给push方法的那些元素，它们的类型为E，因此未受检的转换不会有任何危害</u>。

  ​	**一旦你证明了未受检的转换是安全的，就要在尽可能小的范围中禁止警告（E27）**。在这种情况下，构造器只包含未受检的数组创建，因此可以在整个构造器中禁止这条警告。通过增加一条注解@SuppressWarnings来完成禁止，Stack能够正确无误地进行编译，你就可以使用它了，无须显式的转换，也无须担心会出现 ClassCastException异常:

  ```java
  // The elements array will contain only E instances from push(E).
  // This is sufficient to ensure type safety, but the runtime type of
  // the array won't be E[]; it will always be Object[]!
  @SuppressWarnings("unchecked")
  public Stack() {
    elements = (E[]) new Object[DEFAULT_INITIAL_CAPACITY];
  }
  ```

  ​	**<u>消除Stack中泛型数组创建错误的第二种方法是，将elements域的类型从`E[]`改为`Object[]`</u>**。这么做会得到一条不同的错误：

  ```shell
  Stack.java:19: incompatible types
  found: Object, required: E
  	E result = elements[--size];
  											^
  ```

  ​	通过把数组中获取到的元素由Object转换成E，可以将这条错误变成一条警告：

  ```shell
  Stack.java:19: warning: [unchecked] unchecked cast
  found: Object, required: E
  	E result = (E) elements[--size];
  													^
  ```

  ​	<u>由于**E是一个不可具体化的(non-reifiable)类型，编译器无法在运行时检验转换**。你还是可以自己证实未受检的转换是安全的，因此可以禁止该警告</u>。根据（E27）的建议，我们只要在包含未受检转换的任务禁止警告，而不是在整个pop方法上禁止就可以了，方法如下：

  ```java
  // Appropriate suppression of unchecked warning
  public E pop() {
    if (size == 0)
      throw new EmptyStackException();
   	
    // push requires elements to be of type E, so cast is correct
    @SuppressWarnings("unchecked") E result =  (E) elements[--size];
    elements[size] = null; // Eliminate obsolete reference
    return result;
  }
  ```

  ​	这两种消除泛型数组创建的方法，各有所长。

  + 第一种方法的可读性更强：数组被声明为`E[]`类型清楚地表明它只包含E实例。它也更加简洁：在一个典型的泛型类中，可以在代码中的多个地方读取到该数组；**第一种方法只需要转换一次(创建数组的时候)**。
  + 而**第二种方法则是每次读取一个数组元素时都需要转换一次**。

  ​	因此，**第一种方法优先，在实践中也更常用**。但是，它会导致**<u>堆污染(heap pollution)，详见（E32）：数组的运行时类型与它的编译时类型不匹配(除非E正好是Object)</u>**。这使得有些程序员会觉得很不舒服，因而选择第二种方案，<u>虽然堆污染在这种情况下并没有什么危害</u>。

  ​	下面的程序示范了泛型Stack类的使用方法。程序以倒序的方式打印出它的命令行参数，并转换成大写字母。如果要在从堆栈中弹出的元素上调用String的toUpperCase方法，并不需要显式的转换，并且确保自动生成的转换会成功：

  ```java
  // Little program to exercise our generic Stack
  public static void main(String[] args) {
    Stack<String> stack = new Stack<>();
    for(String arg : args)
      stack.push(arg);
    while(!stack.isEmpty())
      System.out.println(stack.pop().toUpperCase());
  }
  ```

  ​	看来上述的示例与（E28）相矛盾了，（E28）鼓励优先使用List而非数组。实际上不可能总是或者总想在泛型中使用List。**Java并不是生来就支持List，因此有些泛型如 ArrayList必须在数组上实现。为了提升性能，其他泛型如 HashMap也在数组上实现**。

  ​	绝大多数泛型就像我们的 Stack示例一样，因为它们的类型参数没有限制：你可以创建`Stack<Object>`、 `Stack<int[]>`、`Stack<List<String>>`，或者任何其他对象引用类型的Stack。注意不能创建基本类型的 Stack：企图创建`Stack<int>`或者`Stack<double>`会产生一个编译时错误。这是Java泛型系统的一个基本局限性。你可以通过使用基本包装类型(boxed primitive type)来避开这条限制（E61）。

  ​	有一些泛型限制了可允许的类型参数值。例如，以`java.util.concurrent.DelayQueue`为例，其声明内容如下:

  ​	`class DelayQueue<E extends Delayed> implements BlockingQueue<E>`

  ​	类型参数列表(`<E extends Delayed>`)要求实际的类型参数`E`必须是`java.util.concurrent.Delayed`的一个子类型。它允许DelayQueue实现及其客户端在DelayQueue的元素上利用Delayed方法，无须显式的转换，也没有出现ClassCastException的风险。**类型参数E被称作有限制的类型参数(bounded type parameter)**。注意，子类型关系确定了，**每个类型都是它自身的子类型**[JLS，4.10]，因此创建`DelayQueue<Delayed>`是合法的。

---

+ 小结

  ​	总而言之，使用泛型比使用需要在客户端代码中进行转换的类型来得更加安全，也更加容易。<u>在设计新类型的时候，要确保它们不需要这种转换就可以使用。这通常意味着要把类做成是泛型的。只要时间允许，就把现有的类型都泛型化</u>。这对于这些类型的新用户来说会变得更加轻松，又不会破坏现有的客户端（E26）。

## * E30 优先考虑泛型方法

+ 概述

  ​	正如类可以从泛型中受益一般，方法也一样。<u>静态工具方法尤其适合于泛型化</u>。Collections中的所有"算法"方法(例如binarySearch和sort)都泛型化了。

  ​	编写泛型方法与编写泛型类型相类似。例如下面这个方法，它返回两个集合的联合：

  ```java
  // Uses raw types - unacceptable! (E26)
  public static Set union(Set s1, Set s2) {
    Set result = new HashSet(s1);
    result.addAll(s2);
    return result;
  }
  ```

  ​	这个方法可以编译，但是有两条警告：

  ```shell
  Union.java:5: warning: [unchecked] unchecked call to HashSet(Collection<? extends E>) as a member of raw type HashSet
  	Set result = new HashSet(s1);
  								^
  Union.java:6: warning: [unchecked] unchecked call to addAll(Collection<? extends E>) as a member of raw type Set
  	result.addAll(s2);
  								^
  ```

  ​	为了修正这些警告，使方法变成是类型安全的，要将方法声明修改为声明一个类型参数(type parameter)，表示这三个集合的元素类型(两个参数和一个返回值)，并在方法中使用类型参数。

  ​	**声明类型参数的类型参数列表，处在方法的修饰符及其返回值之间**。在这个示例中，类型参数列表为`<E>`，返回类型为`Set<E>`。类型参数的命名惯例与泛型方法以及泛型的相同（E29和E68）：

  ```java
  // Generic method 
  public static <E> Set<E> union(Set<E> s1, Set<E> s2) {
    Set<E> result = new HashSet<>(s1);
    result.addAll(s2);
    return result;
  }
  ```

  ​	至少对于简单的泛型方法而言，就是这么回事了。现在该方法编译时不会产生任何警告，并提供了类型安全性，也更容易使用。以下是一个执行该方法的简单程序。程序中不包含转换，编译时不会有错误或者警告：

  ```java
  // Simple program to exercise generic method
  public static void main(String[] args) {
    Set<String> guys = Set.of("Tom", "Dick", "Harry");
    Set<String> stooges = Set.of("Larry", "Moe", "Curly");
    Set<String> aflCio = union(guys, stooges);
    System.out.println(aflCio);
  }
  ```

  ​	运行这段程序时，会打印出[Moe，Harry，Tom，Curly，Larry，Dick]。(元素的输出顺序是独立于实现的。)

  ​	union方法的局限性在于三个集合的类型(两个输入参数和一个返回值)必须完全相同。<u>利用**有限制的通配符类型**( bounded wildcard type)可以使方法变得更加灵活</u>（E31）。

  ​	有时可能需要创建一个不可变但又适用于许多不同类型的对象。由于泛型是通过**擦除**（E28）实现的，可以给所有必要的类型参数使用单个对象，但是需要编写一个静态工厂方法，让它重复地给每个必要的类型参数分发对象。这种模式称作**泛型单例工厂( generic singleton factory)**，常用于**函数对象**（E42），如`Collections.reverseOrder`，有时也用于像`Collections.emptySet`这样的集合。

  ​	假设要编写一个**恒等函数(identity function)**分发器。类库中提供了`Function.identity`，因此不需要自己编写（E59），但是自己编写也很有意义。如果在每次需要的时候都重新创建一个，这样会很浪费，因为它是无状态的(stateless)。

  ​	<u>如果Java泛型被具体化了，每个类型都需要一个恒等函数，但是它们被擦除后，就只需要一个泛型单例</u>。请看以下示例：

  ```java
  // Generic singleton factory pattern
  private static UnaryOperator<Object> IDENTITY_FN = (t) -> t;
  
  @SuppressWarnings("unchecked")
  public static <T> UnaryOperator<T> identityFunction() {
    return (UnaryOperator<T>) IDENTITY_FN;
  }
  ```

  ​	IDENTITY_FN转换成（`UnaryFunction<T>`），产生了一条未受检的转换警告，因为`UnaryFunction<Object>`对于每个`T`来说并非都是个`Unaryfunction<T>`。但是**恒等函数很特殊：它返回未被修改的参数**，因此我们知道无论的值是什么，用它作为`UnaryFunction<T>`都是类型安全的。因此，我们可以放心地禁止由这个转换所产生的未受检转换警告。一旦禁止，代码在编译时就不会出现任何错误或者警告。

  ​	下面是一个范例程序，它利用泛型单例作为`UnaryFunction<String>`和`UnaryFunction<Number>`。像往常一样，它不包含转换，编译时没有出现错误或者警告：

  ```java
  // Sample program to exercise generic singleton
  public static void main(Stringp[] args) {
    Stringp[] strings = {"jute", "hemp", "nylon"};
    UnaryOperator<String> sameString = identityFunction();
    for(String s : strings)
      System.out.println(sameString.apply(s));
  
    Number[] numbers = { 1, 2.0, 3L };
    UnaryOperator<Number> sameNumber = identityFunction();
    for(Number n : numbers)
      System.out.println(sameNumber.apply(n));
  }
  ```

  ​	虽然相对少见，但是通过某个<u>包含该类型参数本身的表达式来限制类型参数</u>是允许的。这就是**递归类型限制(recursive type bound)**。

  ​	**<u>递归类型限制最普遍的用途与Comparable接口有关，它定义类型的自然顺序（E14）</u>**。这个接口的内容如下：

  ```java
  public interface Comparable<T> {
    int compareTo(T o);
  }
  ```

  ​	类型参数定义的类型，可以与实现`Comparable<T>`的类型的元素进行比较。**实际上，几乎所有的类型都只能与它们自身的类型的元素相比较**。例如String实现`Comparable<String>`，Integer实现`Comparable<Integer>`，等等。

  ​	有许多方法都带有一个实现Comparable接口的元素列表，为了对列表进行排序，并在其中进行搜索，计算出它的最小值或者最大值，等等。要完成这其中的任何一项操作，都要求列表中的每个元素能够与列表中的每个其他元素相比较，换句话说，列表的元素可以互相比较(mutually comparable)。下面是如何表达这种约束条件的一个示例：

  ```java
  // Using a recursive type bound to express mutual comparability
  public static <E extends Comparable<E>> E max(Collectoin<E> c);
  ```

  ​	<u>类型限制`<E extends Comparable<E>>`，可以读作"针对可以与自身进行比较的某个类型E"，这与互比性的概念或多或少有些一致</u>。

  ​	下面的方法就带有上述声明。它根据元素的自然顺序计算列表的最大值，编译时没有出现错误或者警告：

  ```java
  // Returns max value in a collection - uses recursive type bound
  public static <E extends Comparable<E>> E max(Collection<E> c) {
    if(c.isEmpty())
      throw new IllegalArgumentException("Empty collection");
    E result = null;
    for(E e : c)
      if(result == null || e.compareTo(result) > 0)
        result = Objects.requireNonNull(e);
    return result;
  }
  ```

  ​	注意，如果列表为空，这个方法就会抛出IllegalArgumentException异常。更好的替代做法是返回一个 `Optional<E>`（E55）。

  ​	递归类型限制可能比这个要复杂得多，但幸运的是，这种情况并不经常发生。如果你理解了这种习惯用法和它的通配符变量（E31），以及**模拟自类型(simulated self-type)**习惯用法（E2），就能够处理在实践中遇到的许多递归类型限制了。

---

+ 小结

  ​	总而言之，泛型方法就像泛型一样，使用起来比要求客户端转换输入参数并返回值的方法来得更加安全，也更加容易。就像类型一样，你应该确保方法不用转换就能使用，这通常意味着要将它们泛型化。并且就像类型一样，还应该将现有的方法泛型化，使新用户使用起来更加轻松，且不会破坏现有的客户端（E26）。

## * E31 利用有限制通配符来提升API的灵活性

+ 概述

  ​	如（E28）所述，**参数化类型是不变的(invariant)**。换句话说，**对于任何两个截然不同的类型Type1和Type2而言，`List<Type1>`既不是`List<Type2>`的子类型，也不是它的超类型**。

  ​	虽然`List<String>`不是`List<Object>`的子类型，这与直觉相悖，但是实际上很有意义。你可以将任何对象放进一个`List<Object>`中，却只能将字符串放进`List<String>`中。由于`List<String>`不能像`List<Object>`能做任何事情，它不是一个子类型（E10）。

  ​	有时候，我们需要的灵活性要比不变类型所能提供的更多。比如（E29）中的堆栈。下面是它的公共API：

  ```java
  public class Stack<E> {
    public Stack();
    public void push(E e);
    public E pop();
    public boolean isEmpty();
  }
  ```

  ​	假设我们想要增加一个方法，让他按顺序将一系列的元素全部放到堆栈中。第一次尝试如下：

  ```java
  // pushAll method without wildcard type - deficient!
  public void pushAll(Ierable<E> src) {
    for(E e : src)
      push(e);
  }
  ```

  ​	这个方法编译时正确无误，但是并非尽如人意。如果Iterable的src元素类型与堆栈的完全匹配，就没有问题。但是假如有一个`Stack<Number>`，并调用了`push(intVal)`，这里的intVal就是Integer类型。这是可以的，因为Integer是Number的一个子类。因此从逻辑上来说，下面这个方法应该可行：

  ```java
  Stack<Number> numberStack = new Stack<>();
  Iterable<Integer> integers = ... ;
  numberStack.pushAll(integers);
  ```

  ​	但是，如果尝试着么做，就会得到下面的错误信息，<u>因为参数化类型是不可变的</u>：

  ```java
  StackTest.java:7: error: incompatible types: Iterable<Integer>
    cannot be converted to Iterable<Number>
    numberStack.pushAll(integers);
  											^
  ```

  ​	幸运的是，有一种解决办法。Java提供了一种特殊的参数化类型，称作**有限制的通配符类型(bounded wildcard type)**，它可以处理类似的情况。 

  ​	pushAll的输入参数类型不应该为"E的Iterable接口"，而应该为"E的某个子类型的Iterable接口"。

  ​	通配符类型`Iterable<? extends E>`正是这个意思。(使用关键字 extends有些误导：**回忆一下E29中的说法，确定了子类型(subtype)后，每个类型便都是自身的子类型，即便它没有将自身扩展**。)我们修改一下 pushAll来使用这个类型：

  ```java
  // Wildcard type for a parameter that serves as an E producer
  public void pushAll(Iterable<? extends E> src) {
    for(E e : src)
      push(e);
  }
  ```

  ​	修改之后，不仅Stack可以正确无误地编译，没有通过初始的pushAll声明进行编译的客户端代码也一样可以。因为Stack及其客户端正确无误地进行了编译，你就知道一切都是类型安全的了。

  ​	现在假设想要编写一个popAll方法，使之与pushAll方法相呼应。popAll方法从堆栈中弹出每个元素，并将这些元素添加到指定的集合中。初次尝试编写的popAll方法可能像下面这样：

  ```java
  // popAll method without wildcard type - deficient!
  public void popAll(Collection<E> dst) {
    while(!isEmpty())
      dst.add(pop());
  }
  ```

  ​	此外，如果目标集合的元素类型与堆栈的完全匹配，这段代码编译时还是会正确无误，并且运行良好。但是，也并不意味着尽如人意。假设你有一个`Stack<Number>`和Object类型的变量。如果从堆栈中弹出一个元素，并将它保存在该变量中，它的编译和运行都不会出错，那你为何不能也这么做呢?

  ```java
  Stack<Number> numberStack = new Stack<Number>();
  Coleection<Object> objects = ... ;
  numberStack.popAll(objects);
  ```

  ​	如果试着用上述的popAll版本编译这段客户端代码，就会得到一个非常类似于第一次用 pushAll时所得到的错误：`Collection<Object>`不是`Collection<Number>`的子类型。

  ​	这一次通配符类型同样提供了一种解决办法。popAll的输入参数类型不应该为"E的集合"，而应该为"E的某种超类的集合"(这里的超类是确定的，**因此E是它自身的一个超类型**[JLS，4.10])。

  ​	仍有一个通配符类型正符合此意:`Collection<? super E>`。让我们修改popAll来使用它：

  ```java
  // Wildcard type for parameter that serves as an E consumer
  public void popAll(Collection<? super E> dst) {
    while(!isEmpty())
      dst.add(pop());
  }
  ```

  ​	做了这个变动之后， Stack和客户端代码就都可以正确无误地编译了。

  ​	结论很明显：**为了获得最大限度的灵活性，要在表示生产者或者消费者的输入参数上使用通配符类型**。

  ​	<u>如果某个输入参数既是生产者，又是消费者，那么通配符类型对你就没有什么好处了：因为你需要的是严格的类型匹配，这是不用任何通配符而得到的</u>。

  ​	下面的助记符便于让你记住要使用哪种通配符类型：

  ​	**<u>PECS表示 producer-extends， consumer-super</u>**。

  + 换句话说，如果参数化类型表示一个生产者T，就使用`<? extends T>`；
  + 如果它表示一个消费者，就使用`<? super T>`。

  ​	在我们的Stack示例中， pushAll的src参数产生E实例供Stack使用，因此src相应的类型为`Iterable<? extends E>`；popAll的dst参数通过Stack消费E实例，因此dst相应的类型为`Collection<? super E>`。

  ​	<u>PECS这个助记符突出了使用通配符类型的基本原则</u>。 Naftalin和 Wadler称之为 **Get and Put Principle** [Naftalin07， 2. 4]。

  ​	记住这个助记符，下面我们来看一些之前的条目中提到过的方法声明。（E28）中的reduce方法就有这条声明：

  ```java
  public Chooser(Collection<T> choices)
  ```

  ​	这个构造器只用choices集合来生成类型T的值（并把它们保存起来供后续使用），因此它的声明应该用一个`? extends T`的通配符类型。得到的构造器声明如下：

  ```java
  // Wildcard type for parameter that serves as an T producer
  public Chooser(Collection<? extends T> choices)
  ```

  ​	这一变化实际上有什么区别吗？事实上，的确有区别。假设你有一个`List<Integer>`想通过`Function<Number>`把它简化。它不能通过初始声明进行编译，但是一旦添加了有限制的通配符类型，就可以进行编译了。

  ​	现在让我们看看第30条中的 union方法。声明如下:

  ​	`public static <E> Set<E> union(Set<E> s1， Set<E> s2)`

  ​	s1和s2这两个参数都是生产者E，因此根据PECS助记符，这个声明应该是:

  ```java
  public static <E> Set<E> union(Set<? extends E> s1,
                                Set<? extends E> s2)
  ```

  ​	注意返回类型仍然是`Set<E>`。**<u>不要用通配符类型作为返回类型</u>**。除了为用户提供额外的灵活性之外，它还会强制用户在客户端代码中使用通配符类型。修改了声明之后，这段代码就能正确编译了:

  ```java
  Set<Integer> integers = Set.of(1, 3, 5);
  Set<Double> doubles = Set.of(2.0, 4.0, 6.0);
  Set<Number> numbers = union(integers, doubles);
  ```

  ​	如果使用得当，通配符类型对于类的用户来说几乎是无形的。它们使方法能够接受它们应该接受的参数，并拒绝那些应该拒绝的参数。**<u>如果类的用户必须考虑通配符类型，类的API或许就会出错</u>**。

  ​	 <u>在Java8之前，类型推导(type inference)规则还不够智能，它无法处理上述代码片段，还需要编译器使用通过上下文指定的返回类型(或者目标类型)来推断E的类型</u>。

  ​	前面出现过的 union调用的目标类型是`Set<Number>`。如果试着在较早的Java版本中编译这个代码片段(使用`Set.of`工厂相应的替代方法)，将会得到一条像下面这样冗长、繁复的错误消息：

  ```shell
  Union.java:14: error: incompatible types
    Set<Number> numbers = union(integers, doubles);
  															^
    required: Set<Number>
    found:		Set<INT#1>
    where INT#!,INT#2 are in intersection types:
    	INT#1 extends Number, Comparable<? extends INT#2>
    	INT#2 extends Number, Comparable<?>
  ```

  ​	幸运的是，有一种办法可以处理这种错误。**如果编译器不能推断出正确的类型，始终可以通过一个<u>显式的类型参数(explicit type parameter)</u>[JLS，15.12]来告诉它要使用哪种类型**。甚至在Java8中引入目标类型之前，这种情况不经常发生，这是好事，因为显式的类型参数不太优雅。增加了这个显式的类型参数之后，这个代码片段在Java8之前的版本中也能正确无误地进行编译了：

  ```java
  // Explicit type parameter - required prior to Java 8
  Set<Number> numbers = Union.<Number>union(integers, doubles); 
  ```

  ​	接下来，我们把注意力转向（E30）中的max方法。以下是初始的声明：

  ```java
  public static <T extends Comparable<T>> T max(List<T> list)
  ```

  ​	下面是修改过的使用通配符类型的声明：

  ```java
  public static <T extends Comparable<? super T>> T max(List<? extends T> list)
  ```

  ​	为了从初始声明中得到修改后的版本，要应用PECS转换两次。最直接的是运用到参数list。它产生T实例，因此将类型从`List<T>`改成`List<? extends T>`。更灵活的是运用到类型参数T。这是我们第一次见到将通配符运用到类型参数。最初T被指定用来扩展`Comparab1e<T>`，但是T的 comparable消费T实例(并产生表示顺序关系的整值)。因此，参数化类型`Comparable<T>`被有限制通配符类型 `Comparable<? super T>`取代。

  + Comparable始终是消费者，因此**使用时始终应该是`Comparable<? super T>`优先于`Comparable<T>`**。
  + 对于Comparator接口也一样，因此**使用时始终应该是 `Comparator<? super T>`优先于 `Comparator<T>`**。

  ​	修改过的max声明可能是整本书中最复杂的方法声明了。所增加的复杂代码真的起作用了么？是的，起作用了。下面是一个简单的列表示例，在初始的声明中不允许这样，修改过的版本则可以:

  ​	`List<ScheduledFuture<?>> scheduledFutures = ... ;`

  ​	不能将初始方法声明运用到这个List的原因在于，`java.util.concurrent.ScheduledFuture`没有实现 `Comparable<ScheduledFuture>`接口。相反，<u>它是扩展`Comparable<Delayed>`接口的 Delayed接口的子接口。换句话说， ScheduleFuture实例并非只能与其他ScheduledFuture实例相比较；它可以与任何Delayed实例相比较，这就足以导致初始声明时就会被拒绝</u>。**更通俗地说，需要用通配符支持那些不直接实现Comparable(或者 Comparator)而是扩展实现了该接口的类型**。

  ​	还有一个与通配符有关的话题值得探讨。类型参数和通配符之间具有双重性，许多方法都可以利用其中一个或者另一个进行声明。例如，下面是可能的两种静态方法声明，来交换列表中的两个被索引的项目。

  ​	第一个使用无限制的<u>类型参数</u>（E30），第二个使用<u>无限制的通配符</u>：

  ```java
  // Two possible declarations for the swap method
  public static <E> void swap(List<E> list, int i, int j);
  public static void swap(List<?> list, int i , int j);
  ```

  ​	你更喜欢这两种声明中的哪一种呢？为什么？在公共API中，第二种更好一些，因为它更简单。将它传到一个List中（任何List）方法就会交换被索引的元素。不用担心类型参数。

  ​	一般来说，**如果类型参数只在方法声明中出现一次，就可以用通配符取代它**。

  + **如果是无限制的类型参数，就用无限制的通配符取代它；**
  + **如果是有限制的类型参数，就用有限制的通配符取代它。**

  ​	将第二种声明用于swap方法会有一个问题。下面这个简单的实现不能编译

  ```java
  public static void swap(List<?> list, int i, int j) {
    list.set(i, list.set(j, list.get(i)));
  }
  ```

  ​	试着编译时会产生这条没有什么用处的错误消息：

  ```shell
  Swap.java:5: error: incompatible types: Object cannot be converted to CAP#1
  	list.set(i, list.set(j, list.get(i)));
  																	^
  	where CAP#1 is a fresh type-variable:
  		CAP#1 extends Object from capture of ?
  ```

  ​	不能将元素放回到刚刚从中取出的列表中，这似乎不太对劲。<u>问题在于list的类型为`List<?>`，你**不能把null之外的任何值放到`List<?>`中**</u>。

  ​	幸运的是，有一种方式可以实现这个方法，无须求助于不安全的转换或者原生态类型( raw type)。这种想法就是**编写一个私有的辅助方法来捕捉通配符类型**。为了捕捉类型，辅助方法必须是一个泛型方法，像下面这样：

  ```java
  public static void swap(List<?> list, int i, int j) {
    swapHelper(list, i, j);
  }
  
  // Private helper method for wildcard capture
  private static <E> void swapHelper(List<E> list, int i, int j) {
    list.set(i, list.set(j, list.get(i)));
  }
  ```

  ​	**swapHelper方法知道list是一个`List<E>`。因此，它知道从这个列表中取出的任何值均为E类型，并且知道将E类型的任何值放进列表都是安全的**。swap这个有些费解的实现编译起来却是正确无误的。<u>它允许我们导出swap这个比较好的基于通配符的声明，同时在内部利用更加复杂的泛型方法</u>。swap方法的客户端不一定要面对更加复杂的swapHelper声明，但是它们的确从中受益。<u>值得一提的是，辅助方法中拥有的签名，正是我们在公有方法中因为它过于复杂而抛弃的</u>。

---

+ 小结

  ​	总而言之，在API中使用通配符类型虽然比较需要技巧，但是会使API变得灵活得多。如果编写的是将被广泛使用的类库，则一定要适当地利用通配符类型。

  ​	**记住基本的原则：producer-extends， consumer-super(PECS)。**

  ​	**还要记住所有的comparable和comparator都是消费者。**

## * E32 谨慎并用泛型和可变参数

+ 概述

  > [一场由Java堆污染(Heap Pollution)引发的思考_zxm317122667的专栏-CSDN博客](https://blog.csdn.net/zxm317122667/article/details/78400398)	

  ​	可变参数(vararg)方法（E53）和泛型都是在Java5中就有了，因此你可能会期待它们可以良好地相互作用；遗憾的是，它们不能。

  ​	可变参数的作用在于让客户端能够将可变数量的参数传给方法，但这是个技术露底(leaky abstration)：当调用一个可变参数方法时，会创建一个**数组**用来存放可变参数；<u>这个数组应该是一个实现细节，它是可见的</u>。因此，<u>当可变参数有泛型或者参数化类型时，编译警告信息就会产生混乱</u>。

  ​	回顾一下（E28），**<u>非具体化(non-reifiable)类型是指其运行时代码信息比编译时少，并且显然所有的泛型和参数类型都是非具体化的</u>**。

  ​	<u>如果一个方法声明其可变参数为non-reifiable类型，编译器就会在声明中产生一条警告。如果方法是在类型为non-reifiable的可变参数上调用，编译器也会在调用时发出一条警告信息</u>。这个警告信息类似于：

  ```shell
  warning: [unchecked] Possible heap pollution from parameterized varag type List<String>
  ```

  ​	当一个参数化类型的变量指向一个不是该类型的对象时，会产生**堆污染（heap pollution）**[JLS，4.12.2]。它导致编辑器的自动生成转换失败，破坏了泛型系统的基本保证。

  ​	举个例子。下面的代码是对（E28）的代码片段稍做修改得到的：

  ```java
  // Mixing generics and varargs can violate type safety!
  static void dangerous(List<String>... stringLists) {
    List<Integer> intList = List.of(42);
    Object[] objects = stringLists;
    objects[0] = intList;	//	Heap pollution
    String s = stringLists[0].get(0); // ClassCastException
  }
  ```

  ​	这个方法没有可见的转换，但是在调用一个或者多个参数时会抛出ClassCastException异常。上述最后一行代码中有一个不可见的转换，这是由编译器生成的。这个转换失败证明类型安全已经受到了危及，因此**<u>将值保存在泛型可变参数数组参数中是不安全的</u>**。

  ​	这个例子引出了一个有趣的问题：**为什么显式创建泛型数组是非法的，用泛型可变参数声明方法却是合法的呢**？换句话说，为什么之前展示的方法只产生一条警告，而（E28）中的代码片段却产生一个错误呢？答案在于，**<u>带有泛型可变参数或者参数化类型的方法在实践中用处很大，因此Java语言的设计者选择容忍这一矛盾的存在</u>**。

  ​	事实上，Java类库导出了好几个这样的方法，包括`Arrays.aslist(T... a)`、`Collections.addAll(Collection<? super T> C, T... elements)`， 以及`EnumSet.of(E first, E... rest)`。<u>与前面提到的危险方法不一样，这些类库方法是类型安全的</u>。

  ​	在Java7之前，带泛型可变参数的方法的设计者，对于在调用处出错的警告信息一点办法也没有。这使得这些API使用起来非常不愉快。用户必须忍受这些警告，要么最好在每处调用点都通过`@Suppresswarnings(" unchecked")`注解来消除警告（E27）。这么做过于烦琐，而且影响可读性，并且掩盖了反映实际问题的警告。

  ​	<u>在Java7中，增加了@SafeVarargs注解，它让带泛型vararg参数的方法的设计者能够自动禁止客户端的警告</u>。本质上， **@SafeVarargs注解是通过方法的设计者做出承诺，声明这是类型安全的**。作为对于该承诺的交换，编译器同意不再向该方法的用户发出警告说这些调用可能不安全。

  ​	<u>重要的是，不要随意用@Safevarargs对方法进行注解，除非它真正是安全的</u>。那么它凭什么确保安全呢？回顾一下，泛型数组是在调用方法的时候创建的，用来保存可变参数。如果该方法没有在数组中保存任何值，也不允许对数组的引用转义(这可能导致不被信任的代码访问数组)，那么它就是安全的。**<u>换句话说，如果可变参数数组只用来将数量可变的参数从调用程序传到方法(毕竞这才是可变参数的目的)，那么该方法就是安全的</u>**。

  ​	<u>**值得注意的是，从来不在可变参数的数组中保存任何值，这可能破坏类型安全性**</u>。以下面的泛型可变参数方法为例，它返回了一个包含其参数的数组。乍看之下，这似乎是一个方便的小工具：

  ```java
  // UNSAFE - Exposes a reference to its generic parameter array!
  static <T> T[] toArray(T... args) {
    return args;
  }
  ```

  ​	这个方法只是返回其可变参数数组，看起来没什么危险，但它实际上很危险！<u>这个数组的类型，是由传到方法的参数的**编译时类型**来决定的，编译器没有足够的信息去做准确的决定</u>。因为该方法返回其可变参数数组，它会将堆污染传到调用堆栈上。

  ​	下面举个具体的例子。这是一个泛型方法，它带有三个类型为T的参数，并返回一个包含两个(随机选择的)参数的数组：

  ```java
  static<T> T[] pickTwo(T a, T b, T c) {
    switch(ThreadLocalRandom.current().nextInt(3)) {
      case 0: return toArray(a, b);
      case 1: return toArray(a, c);
      case 2: return toArray(b, c);
    }
    throw new AssertionError(); // Can't get here
  }
  ```

  ​	这个方法本身并没有危险，也不会产生警告，除非它调用了带有泛型可变参数的toArray方法。

  ​	<u>在编译这个方法时，编译器会产生代码，创建一个可变参数数组，并将两个T实例传到toArray。这些代码配置了一个类型为`Object[]`的数组，这是确保能够保存这些实例的最具体的类型，无论在调用时给 pickTwo传递什么类型的对象都没问题。 toArray方法只是将这个数组返回给pickTwo，反过来也将它返回给其调用程序，因此 pickTwo始终都会返回一个类型为`Object[]`的数组</u>。

  ​	现在以下面的main方法为例，练习一下 pickTwo的用法：

  ```java
  public static void main(String[] args) {
    String[] attributes = pickTwo("Good", "Fast", "Cheap");
  }
  ```

  ​	这个方法压根没有任何问题，因此编译时不会产生任何警告。但是在运行的时候，它会抛出一个ClassCastException，虽然它看起来并没有包括任何的可见的转换。<u>你看不到的是，编译器在 pickTwo返回的值上产生了一个隐藏的`String[]`转换</u>。但转换失败了，这是因为从实际导致堆污染(toArray)的方法处移除了两个级别，可变参数数组在实际的参数存入之后没有进行修改。

  ​	这个范例是为了告诉大家，<u>**允许另一个方法访问一个泛型可变参数数组是不安全的**，有两种情况例外</u>：

  + **将数组传给另一个用`@SafeVarargs`正确注解过的可变参数方法是安全的；**
  + **将数组传给只计算数组内容部分函数的非可变参数方法也是安全的**。

  ​	这里有一个安全使用泛型可变参数的典型范例。这个方法中带有一个任意数量参数的列表，并按顺序返回包含输入清单中所有元素的唯一列表。由于该方法用 @SafeVarargs注解过，因此在声明处或者调用处都不会产生任何警告：

  ```java
  // Safe method with a generic varargs parameter
  @SafeVarags
  static <T> List<T> flatten(List<? extends T>... lists) {
    List<T> result = new ArrayList<>();
    for(List<? extends T> list : lists)
      result.addAll(list);
    return result;
  }
  ```

  ​	确定何时应该使用@SafeVarargs注解的规则很简单：**<u>对于每一个带有泛型可变参数或者参数化类型的方法，都要用@SafeVarargs进行注解</u>**，这样它的用户就不用承受那些无谓的、令人困惑的编译警报了。

  ​	这意味着应该永远都不要编写像dangerous或者toArray这类不安全的可变参数方法。<u>每当编译器警告你控制的某个带泛型可变参数的方法可能形成堆污染，就应该检查该方法是否安全</u>。

  ​	这里先提个醒，**泛型可变参数方法在下列条件下是安全的**：

  ​	1、它没有在可变参数数组中保存任何值。

  ​	2、它没有对不被信任的代码开放该数组(或者其克隆程序)。

  ​	以上两个条件只要有任何一条被破坏，就要立即修正它。

  ​	<u>注意，@SafeVarargs注解只能用在无法被覆盖的方法上，因为它不能确保每个可能的覆盖方法都是安全的</u>。<u>在Java8中，该注解只在静态方法和 final实例方法中才是合法的；在Java9中，它在私有的实例方法上也合法了</u>。

  ​	如果不想使用@SafeVarargs注解，也可以采用（E28）的建议，**用一个List参数代替可变参数**(这是一个伪装数组)。下面举例说明这个办法在 flatten方法上的运用。注意，此处只对参数声明做了修改：

  ```java
  // List as a typesafe alternative to a generic varargs parameter
  static <T> List<T> flattern(List<List<? extends T>> lists) {
    List<T> result = new ArrayList<>();
    for(List<? extends T> list : lists)
      result.addAll(list);
    return result;
  }
  ```

  ​	随后，这个方法就可以结合静态工厂方法`List.of`一起使用了，允许使用数量可变的参数。注意，使用该方法的前提是用`@SafeVarargs`对`List.of`声明进行了注解：

  ```java
  audience = flatten(List.of(friends, romans, countrymen));
  ```

  ​	这种做法的优势在于编译器可以证明该方法是类型安全的。你不必再通过@SafeVarargs注解来证明它的安全性，也不必担心自己是否错误地认定它是安全的。其缺点在于客户端代码有点烦琐，运行起来速度会慢一些。

  ​	这一技巧也适用于无法编写出安全的可变参数方法的情况，比如本条之前提到的toArray方法。其List对应的是List.of方法，因此我们不必编写；Java类库的设计者已经替我们完成了。因此pickTwo方法就变成了下面这样：

  ```java
  static<T> List<T> pickTwo(T a, T b, T c) {
    switch(rnd.nextInt(3)) {
      case 0: return List.of(a, b);
      case 1: return List.of(a, c);
      case 2: return List.of(b, c);
    }
    throw new AssertionError();
  }
  ```

  main方法变成如下：

  ```java
  public static void main(String[] args) {
    List<String> attributes = pickTwo("Good", "Fast", "Cheap");
  }
  ```

  ​	<u>这样得到的代码就是类型安全的，因为它只使用泛型，没有用到数组</u>。

---

+ 小结

  ​	总而言之，可变参数和泛型不能良好地合作，这是因为可变参数设施是构建在顶级数组之上的一个技术露底，泛型数组有不同的类型规则。

  ​	虽然泛型可变参数不是类型安全的，但它们是合法的。<u>如果选择编写带有泛型(或者参数化)可变参数的方法，首先要确保该方法是类型安全的，然后用@SafeVarargs对它进行注解，这样使用起来就不会出现不愉快的情况了</u>。

## * E33 优先考虑类型安全的异构容器

+ 概述

  ​	泛型最常用于集合，如`Set<E>`和`Map<K, V>`，以及单个元素的容器，如`ThreadLocal<T>`和 `AtomicReference<T>`。在所有这些用法中，它都<u>充当被参数化了的容器</u>。这样就限制每个容器只能有固定数目的类型参数。一般来说，这种情况正是你想要的。一个Set只有一个类型参数，表示它的元素类型；一个Map有两个类型参数，表示它的键和值类型......

  ​	但是，有时候你会需要更多的灵活性。例如，数据库的行可以有任意数量的列，如果能以类型安全的方式访问所有列就好了。幸运的是，有一种方法可以很容易地做到这一点。这种方法就是<u>将键(key)进行参数化而不是将容器(container)参数化</u>。然后将参数化的键提交给容器来插入或者获取值。用泛型系统来确保值的类型与它的键相符。

  ​	下面简单地示范一下这种方法：以 Favorites类为例，它允许其客户端从任意数量的其他类中，保存并获取一个"最喜爱"的实例。Class对象充当参数化键的部分。之所以可以这样，是因为**类Class被泛型化了。类的类型从字面上来看不再只是简单的Class，而是`Class<T>`**。例如，String.class属于`Class<String>`类型， Integer.class属于`Class<Integer>`类型。<u>当一个类的字面被用在方法中，来传达编译时和运行时的类型信息时，就被称作**类型令牌(type token)**</u>[ Brancha04]

  ​	Favorites类的API很简单。它看起来就像一个简单的映射，除了键(而不是映射)被参数化之外。客户端在设置和获取最喜爱的实例时提交Class对象。下面就是这个API:

  ```java
  // Typesafe heterogeneous container pattern - API
  public class Favorites {
  	public <T> void putFavorite(Class<T> type, T instance);
    public <T> T getFavorite(Class<T> type);
  }
  ```

  ​	下面是一个示例程序，检验一下Favorites类，它将保存、获取并打印一个最喜爱的String、Integer和Class实例：

  ```java
  // Typesafe heterogeneous container pattern - client
  public static void main(String[] args) {
    Favories f = new Favorites();
    f.putFavorite(String.class, "Java");
    f.putFavorite(Integer.class, 0xcafebabe);
    f.putFavorite(Class.class, Favorites.class);
    String favoriteString = f.getFavorite(String.class);
    int favoriteInteger = f.getFavorite(Integer.class);
    Class<?> favoriteClass = f.getFavorite(Class.class);
    System.out.printf("%s %x %s%n", favoriteString, favoriteInteger, favoriteClass.getName());
  }
  ```

  ​	正如所料，这段程序打印出的是 Java cafebabe Fsavorites。注意，有时Java的printf方法与C语言中的不同，C语言中使用`\n`的地方，在Java中应该使用`%n`。这个`%n`会产生适用于特定平台的行分隔符，在许多平台上是`\n`，但是并非所有平台都是如此。

  ​	<small>ps：(自己java试了下printf，`\n`或者`%n`都会换行)</small>

  + Favorites实例是<u>**类型安全(typesafe)**的：当你向它请求String的时候，它从来不会返回一个Integer给你</u>。
  + 同时它也是<u>**异构的(heterogeneous)**：不像普通的映射，它的所有键都是不同类型的</u>。

  ​	因此，我们将 Favorites称作**类型安全的异构容器**(typesafe heterogeneous container)

  ​	Favorites的实现小得出奇。它的完整实现如下

  ```java
  // Typesafe heterogeneous container pattern - implementation
  public class Favorites {
    private Map<Class<?>, Object> favorites = new HashMap<>();
    
    public <T> void putFavorite(Class<T> type, T instance) {
      favorites.put(Objects.requireNonNull(type), instance);
    }
    
    public <T> T getFavorite(Class<T> type) {
      return type.cast(favorites.get(type));
    }
  }
  ```

  ​	这里面发生了一些微妙的事情。每个Favorites实例都得到一个称作 favorites的私有`Map<Class<?>,Object>`的支持。你可能认为由于无限制通配符类型的关系，将不能把任何东西放进这个Map中，但事实正好相反。<u>要注意的是通配符类型是嵌套的：它不是属于通配符类型的Map的类型，而是它的键的类型</u>。

  ​	由此可见，每个键都可以有一个不同的参数化类型：一个可以是`Class<String>`，接下来是`Class<Integer>`等。异构就是从这里来的。

  ​	第二件要注意的事情是，favorites Map的值类型只是Object。换句话说，Map并不能保证键和值之间的类型关系，即不能保证每个值都为它的健所表示的类型(通俗地说，就是指键与值的类型并不相同——译者注)。事实上，Java的类型系统还没有强大到足以表达这一点。但我们知道这是事实，并在获取favorite的时候利用了这一点。

  ​	putFavorite方法的实现很简单：它只是把(从指定的Class对象到指定的favorite实例)一个映射放到 favorites中。如前所述，这是放弃了键和值之间的"类型联系"因此无法知道这个值是键的一个实例。但是没关系，因为getFavorites方法能够并且的确重新建立了这种联系。

  ​	<u>getFavorite方法的实现比putFavorite的更难一些。它先从favorites映射中获得与指定 Class对象相对应的值。这正是要返回的对象引用，但它的编译时类型是错误的。它的类型只是 Object( favorites映射的值类型)，我们需要返回一个T。因此，**getfavorite方法的实现利用Class的cast方法，将对象引用动态地转换( dynamically cast)成了Class对象所表示的类型**</u>。

  ​	cast方法是Java的转换操作符的动态模拟。它只检验它的参数是否为Class对象所表示的类型的实例。如果是，就返回参数；否则就抛出ClassCastException异常。我们知道getFavorite中的cast调用永远不会抛出 ClassCastException异常，并假设客户端代码正确无误地进行了编译。也就是说，我们知道favorites映射中的值会始终与键的类型相匹配。

  ​	假设cast方法只返回它的参数，那它能为我们做什么呢？**<u>cast方法的签名充分利用了Class类被泛型化的这个事实。它的返回类型是Class对象的类型参数</u>**：

  ```java
  public class Class<T> {
    T cast(Object obj);
  }
  
  // 完整的源代码如下
  public T cast(Object obj) {
    if (obj != null && !isInstance(obj))
      throw new ClassCastException(cannotCastMsg(obj));
    return (T) obj;
  }
  ```

  ​	这正是 getFavorite方法所需要的，也正是让我们不必借助于未受检地转换成T就能确保Favorites类型安全的东西。

  ​	Favorites类有两种局限性值得注意。首先，<u>恶意的客户端可以很轻松地破坏Favorites实例的类型安全，只要以它的**原生态形式**(raw form)使用Class对象</u>。但会造成客户端代码在编译时产生未受检的警告。这与一般的集合实现，如 HashSet和 HashMap并没有什么区别。你可以很容易地利用原生态类型HashSet（E26）将 String放进`HashSet<Integer>`中。也就是说，如果愿意付出一点点代价，就可以拥有运行时的类型安全。<u>确保Favorites永远不违背它的类型约束条件的方式是，让putFavorite方法检验 instance是否真的是type所表示的类型的实例。只需使用一个**动态的转换**</u>，如下代码所示：

  ```java
  // Achieving runtime type safety with a dynamic cast 
  public <T> void putFavorite(Class<T> type, T instance) {
    favorites.put(type, type.cast(instance));
  }
  ```

  ​	<u>`java.util.Collections`中有一些集合包装类采用了同样的技巧。它们称作checkedSet、 checkList、 checkedMap，诸如此类</u>。除了一个集合(或者映射)之外，它们的静态工厂还采用一个(或者两个)Class对象。静态工厂属于泛型方法，**确保Class对象和集合的编译时类型相匹配**。包裝类给它们所封装的集合增加了具体化。例如，如果有人试图将Coin放进你的`Collection<Stamp>`，包装类就会在运行时抛出 ClassCastException异常。用这些包装类在混有泛型和原生态类型的应用程序中追溯"是谁把错误的类型元素添加到了集合中"很有帮助。
  
  ​	Favorites类的第二种局限性在于它<u>不能用在不可具体化的(non-reifiable)类型中（E28）</u>。换句话说，你可以保存最喜爱的String或者String[]，但不能保存最喜爱的`List<String>`。如果试图保存最喜爱的`List<String>`，程序就不能进行编译。原因在于**<u>你无法为`List<String>`获得一个Class对象</u>**：`List<String>.class`是个语法错误，这也是件好事。**`List<String>`和`List<Integer>`共用一个Class对象，即`List.class`**。
  
  ​	<u>如果从"类型的字面"(type literal)上来看，`List<String>.class`和`List<Integer>.class`是合法的，并返回了相同的对象引用，这会破坏 Favorites对象的内部结构。对于这种局限性，还没有完全令人满意的解决办法</u>。
  
  ​	Favorites使用的类型令牌(type token)是无限制的：getFavorite和 putFavorite接受任何Class对象。有时可能需要限制那些可以传给方法的类型。这可以通过有限制的类型令牌( bounded type token)来实现，它只是一个类型令牌，利用有限制类型参数（E30）或者有限制通配符（E31），来限制可以表示的类型。
  
  ​	**注解API（E39）广泛利用了有限制的类型令牌**。
  
  ​	例如，这是一个在运行时读取注解的方法。这个方法来自 AnnotatedElement接口，它通过表示类、方法、域及其他程序元素的反射类型来实现：
  
  ```java
  public <T extends Annotation> T getAnnotation(Class<T> annotationType);
  ```
  
  ​	<u>参数annotationType是一个表示注解类型的有限制的类型令牌</u>。如果元素有这种类型的注解，该方法就将它返回；如果没有，则返回null。**<u>被注解的元素本质上是个类型安全的异构容器，容器的键属于注解类型</u>**。
  
  ​	假设你有一个类型为`Class<?>`的对象，并且想将它传给一个需要有限制的类型令牌的方法，例如 getAnnotation。你可以将对象转换成`Class<? extends Annotation>`，但是这种转换是非受检的，因此会产生一条编译时警告（E27）。幸运的是，类Class提供了一个安全(且动态)地执行这种转换的实例方法。该方法称作<u>asSubclass，它将调用它的Class对象转换成用其参数表示的类的一个子类。如果转换成功，该方法返回它的参数；如果失败，则抛出ClassCastException异常</u>。
  
  ​	下面示范如何**利用asSubclass方法在编译时读取类型未知的注解**。这个方法编译时没有出现错误或者警告：
  
  ```java
  // Use of asSubclass to safety cast to a bounded type token
  static Annotation getAnnotation(AnnotatedElement element, String annotationTypeName) {
    Class<?> annotationType = null; // Unbounded type token
    try {
      annotationType = Class.forName(annotationTypeName);
    } catch (Exception ex) {
      throw new IllegalArgumentException(ex);
    }
    return element.getAnnotation(annotationType.asSubclass(Annotation.class));
  }
  ```
  
  > asSubclass方法的源码如下：
  >
  > ```java
  > // Class.java
  > 
  > @SuppressWarnings("unchecked")
  > public <U> Class<? extends U> asSubclass(Class<U> clazz) {
  >   if (clazz.isAssignableFrom(this))
  >     return (Class<? extends U>) this;
  >   else
  >     throw new ClassCastException(this.toString());
  > }
  > ```

---

+ 小结

  ​	总而言之，**集合API说明了泛型的一般用法，限制每个容器只能有固定数目的类型参数。你可以通过<u>将类型参数放在键上而不是容器上</u>来避开这一限制**。

  ​	对于这种**<u>类型安全的异构容器</u>**，可以用 Class对象作为键。以这种方式使用的Class对象称作**<u>类型令牌</u>**。你也可以使用定制的键类型。例如，用一个 DatabaseRow类型表示一个数据库行(容器)，用泛型`Column<T>`作为它的键。

