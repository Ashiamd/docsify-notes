# 《Effective-Java第三版》-读书笔记-02

> + `1、`、`2、`表示：第1章、第2章
>
> + `E1`、`E2`表示：第1条、第2条
> + `* X、`表示：个人认为第X章是重点（注意点）
> + `* EX`表示：个人认为第X条是重点（注意点）
>
> 全书共12章，90条目
>
> 下面提到的“设计模式”，指《Design Patterns - Elements of Reusable Object-Oriented Software》一书中提到的23种设计模式

# 6、枚举和注解

​	Java支持两种特殊用途的引用类型：一种是类，称作枚举类型(enum type)；一种是接口，称作注解类型(annotation type)。本章将讨论这两个新类型的最佳使用实践。

## E34 用enum代替int常量

> [Java 静态内部类的加载时机 - は問わない - 博客园 (cnblogs.com)](https://www.cnblogs.com/zouxiangzhongyan/p/10762540.html)

+ 引入enum之前的方案

  ​	在Java引入枚举类型之前，通常使用一组int常量来表示枚举类型，其中每一个int常量表示枚举类型的一个成员。但是int枚举模式，不具类型安全性，也几乎没有描述性可言。

  ​	采用int枚举模式的程序是十分脆弱的。因为**int枚举是编译时常量(constant variable)** [JLS，4.12.4]，它们的int值会被编译到使用它们的客户端中。如果与int枚举常量关联的值发生了变化，客户端必须重新编译。如果没有重新编译，客户端程序还是可以运行，不过其行为已经不再准确。

  ​	*很难将int枚举常量转换成可打印的字符串。就算将这种常量打印岀来，或者从调试器中将它显示出来，你所见到的也只是一个数字，这几乎没有什么用处。当需要遍历一个int枚举模式中的所有常量，以及获得int枚举数组的大小时，在int枚举模式中，几乎不存在可靠的方式。*

  ​	另一种类似的即String枚举模式（String enum pattern），同样效果不是很理想。

---

+ enum概述

  ​	**Java的枚举类型的本质是int值**。

  ​	Java枚举类型的基本想法非常简单：这些类通过**公有的静态final域为每个枚举常量导出一个实例**。**<u>枚举类型没有可以访问的构造器，所以它是真正的final类</u>**。客户端不能创建枚举类型的实例，也不能对它进行扩展，因此不存在实例，而只存在声明过的枚举常量。

  ​	换句话说，枚举类型是实例受控的（E6）。它们是单例(Singleton)（E3）的泛型化，本质上是单元素的枚举。

  ​	**枚举类型保证了编译时的类型安全**。例如声明参数的类型为Apple，它就能保证传到该参数上的任何非空的对象引用一定属于三个有效的Apple值之一，而其他任何试图传递类型错误的值都会导致编译时错误，就像<u>试图将某种枚举类型的表达式赋给另一种枚举类型的变量，或者试图利用\=\=操作符比较不同枚举类型的值都会导致编译时错误</u>。

  ​	**包含同名常量的多个枚举类型可以在一个系统中和平共处，因为每个类型都有自己的<u>命名空间</u>**。你可以增加或者重新排列枚举类型中的常量，而无须重新编译它的客户端代码，因为导出常量的域在枚举类型和它的客户端之间提供了一个隔离层：<u>常量值并没有被编译到客户端代码中，而是在int枚举模式之中。最终，可以通过调用toString方法，将枚举转换成可打印的字符串</u>。	

  ​	除了完善int枚举模式的不足之外，枚举类型还允许添加任意的方法和域，并实现任意的接口。它们提供了所有 Object方法（E3）的高级实现，实现了 Comparable（E14）和 Serializable接口(详见第12章)，并针对枚举类型的可任意改变性设计了序列化方式。

  ​	**为了将数据与枚举常量关联起来，得声明实例域，并编写一个带有数据并将数据保存在域中的构造器**。枚举天生就是不可变的，因此所有的域都应该为final的（E17）。它们可以是公有的，但最好将它们声明为私有的，并提供公有的访问方法（E16）。

  ​	**所有的枚举，都有一个静态的values方法，按照声明顺序返回它的值数组。toString方法返回每个枚举值的声明名称，使得println和printf的打印变得更加容易**。

  ​	如果将一个元素从一个枚举类型移除，那么没有引用该元素的任何客户端程序都会继续正常工作。<u>如果客户端引用了被删除的元素，重新编译客户端就会失败，并在引用被删除的枚举元素的那一条出现一条错误信息；如果没有重新编译客户端代码，在运行时就会在这一行抛出一个异常。这是你能期待的最佳行为了，远比使用int枚举模式时要好得多</u>。

  ​	有些与枚举常量相关的行为，可能只会用在枚举类型的定义类或者所在的包中，那么这些方法最好被实现成私有的或者包级私有的。于是每个枚举常量都带有一组隐藏的行为，这使得枚举类型的类或者所在的包能够运作得很好，**像其他的类一样，除非要将枚举方法导出至它的客户端，否则都应该声明为私有的，或者声明为包级私有的（E15）**。

  ​	<u>如果一个枚举具有普遍适用性，它就应该成为一个顶层类(top-level class)；如果它只是被用在一个特定的顶层类中，它就应该成为该顶层类的一个成员类（E24）</u>。例如，`java.math.RoundingMode`枚举表示十进制小数的舍入模式(rounding mode)。这些舍入模式被用于BigDecimal类，但是它们却不属于BigDecimal类的一个抽象。通过使RoundingMode变成一个顶层类，库的设计者鼓励任何需要舍入模式的程序员重用这个枚举，从而增强API之间的一致性。

  ​	**枚举类型中的抽象方法必须被它的所有常量中的具体方法所覆盖**。

  ​	<u>枚举类型有一个自动产生的`valueOf(String)`方法，它将常量的名字转成常量本身。如果在枚举类型中覆盖toString方法，要考虑编写一个fromString方法，将定制的字符串表示法变回相应的枚举</u>。

  ​	**除了编译时常量域（E34）之外，枚举构造器不可以访问枚举的静态域**。这一限制是有必要的，<u>因为构造器运行的时候，这些静态域还没有被初始化。这条限制只有一个特例：枚举常量无法通过构造器访问另一个构造器</u>。

  ​	那么什么时候应该使用枚举呢？**每当需要一组固定常量，并且在编译时就知道其成员的时候，就应该使用枚举**。当然，这包括"天然的枚举类型"，例如行星、一周的天数以及棋子的数目等。但它也包括你在编译时就知道其所有可能值的其他集合，例如菜单的选项、操作代码以及命令行标记等。**枚举类型中的常量集并不一定要始终保持不变**。专门设计枚举特性是考虑到枚举类型的二进制兼容演变。

---

+ 代码举例

  ​	下面代码可以为任何枚举完成一个String到枚举的转化，只要每个常量都有一个独特的字符串表示法：

  ```java
  // Implementing a fromString method on an enum type
  private static final Map<String, Operation> stringToEnum = Stream.of(values()).collect(toMap(Object::toString, e -> e));
  
  // Return Operation for string, if any
  public static Optional<Operation> fromString(String symbol) {
    return Optional.ofNullable(stringToEnum.get(symbol));
  }
  ```

  ​	注意返回`Optional<Operation>`的fromString方法。它用该方法表明：传入的字符串并不代表一项有效的操作，并强制客户端面对这种可能性（E55）。

  ---

  ​	**策略枚举（strategy enum）**代码举例：

  ```java
  // The strategy enum pattern
  enum PayrollDay {
    MONDAY, TUESDAY, WEDNESDAY, THUSDAY, FRIDAY,
    SATURDAY(PayType.WEEKEND), SUNDAY(PayType.WEEDEND);
    
    private final PayType payType;
    
    PayrollDay(PayType payType) { this.payType = payType; }
    PayrollDay() { this(PayType.WEEKDAY); } // Default
    
    int pay(int minutesWorked, int payRate) {
      return payType.pay(minutesWorked, payRate);
    }
    
    // The strategy enum type
    private enum PayType {
      WEEKDAY {
        int overtimePay(int minsWorked, int payRate) {
          return minsWorked <= MINS_PER_SHIFT ? 0 :
          (minsWorked - MINS_PER_SHIFT) * payRate / 2;
        }
      },
      WEEKEND {
  			int overtimePay(int minsWorked, int payRate) {
          return minsWorked * payRate / 2;
        }
      };
      
      abstract int overtimePay(int mins, int payRate);
      
      private static final int MINS_PER_SHIFT = 8 * 60;
      
      int pay(int minsWorked, int payRate) {
        int basePay = minsWorked * payRate;
        return baseaPay + overtimePay(minsWorked, payRate);
      }
    }
  }
  ```

  ​	如果枚举中的switch语句不是在枚举中实现特定于常量的行为的一种很好的选择，那么它们还有什么用处呢？**枚举中的switch语句适合于给外部的枚举类型增加特定于常量的行为**。例如，假设 Operation枚举不受你的控制，你希望它有一个实例方法来返回每个运算的反运算。你可以用下列静态方法模拟这种效果：

  ```java
  // Switch on an enum to simulate a missing method
  public static Operation inverse(Operation op) {
    switch(op) {
      case PLUS: return Operation.MINUS;
      case MINUS: return Operation.PLUS;
      case TIMES: return Operation.DIVIDE;
      case DIVIDE: return Operation.TIMES;
      default: throw new AssertionError("Unknown op: " + op);
    }
  }
  ```

  > ​	一般来说，枚举通常在性能上与int常量相当。与int常量相比，枚举有个小小的性能缺点，即装载和初始化枚举时会需要空间和时间的成本，但在实践中几乎注意不到这个问题。

---

+ 小结

  ​	总而言之，与int常量相比，枚举类型的优势是不言而喻的。枚举的可读性更好，也更加安全，功能更加强大。

  ​	许多枚举都不需要显式的构造器或者成员，但许多其他枚举则受益于属性与每个常量的关联以及其行为受该属性影响的方法。

  ​	只有极少数的枚举受益于将多种行为与单个方法关联。在这种相对少见的情况下，特定于常量的方法要优先于启用自有值的枚举。

  ​	<u>如果多个(但非所有)枚举常量同时共享相同的行为，则要考虑策略枚举</u>。

## E35 用实例域代替序数

+ 概述

  ​	许多枚举天生就与一个单独的int值相关联。所有的枚举都有一个ordinal方法，它返回每个枚举常量在类型中的数字位置。你可以试着从序数中得到关联的int值：

  ```java
  // Enum.java 源码
  public abstract class Enum<E extends Enum<E>>
    implements Comparable<E>, Serializable { 
  	
    // ... 省略其他代码
    private final int ordinal;
  
    /**
       * Returns the ordinal of this enumeration constant (its position
       * in its enum declaration, where the initial constant is assigned
       * an ordinal of zero).
       *
       * Most programmers will have no use for this method.  It is
       * designed for use by sophisticated enum-based data structures, such
       * as {@link java.util.EnumSet} and {@link java.util.EnumMap}.
       *
       * @return the ordinal of this enumeration constant
       */
    public final int ordinal() {
      return ordinal;
    }
    
    // ... 省略其他代码
  }
  ```

  ​	**永远不要根据枚举的序数导出与它关联的值，而是要将它保存在一个实例域中**：

  ```java
  public enum Ensemble {
  	SOLO(1), DUET(2); // 比较懒，这里省略了余下枚举实例
    private final int numberOfMusicians;
    Ensemble(int size) { this.numberOfMusicians = size; }
    public int numberOfMusicians() { return numberOfMusicians; }
  }
  ```

  ​	Enum规范中谈及ordinal方法时写道："大多数程序员都不需要这个方法。它是设计用于像 EnumSet和 EnumMap这种基于枚举的通用数据结构的。" <u>除非你在编写的是这种数据结构，否则**最好完全避免使用ordinal方法**</u>。

## E36 用EnumSet代替位域

+ 概述

  ​	如果一个枚举类型的元素主要用在集合中，一般就使用int枚举模式（E34），比如将2的不同倍数赋予每个常量：

  ```java
  // Bit field enumeration constants - OBSOLETE!
  public class Text {
    public static final int STYLE_BOLD  = 1 << 0; // 1
    public static final int STYLE_ITALIC = 1 << 1; // 2
    public static final int STYLE_UNDERLINE = 1 << 2; // 4
    public static final int STYLE_STRIKETHROUGH = 1 << 3; // 8
    
    // Parameter is bitwise OR of zero or more STYLE_ constants
    public void applyStyles(int styles) { ... }
  }
  ```

---

+ 位域

  ​	通过OR位运算将几个常量合并到一个集合中，称作位域（bit field）：

  ```java
  text.applyStyles(STYLE_BOLD | STYLE_ITALIC);
  ```

  优势：

  + 有效地执行像union（求并集）和intersection（求交集）的集合操作。

  缺点：

  + 具有int枚举常量的所有缺点。
  + 翻译位域比翻译简单的int枚举常量困难。
  + 遍历元素困难。
  + 需要预测最多需要多少位，选择相应类型（一般是int或者long）。

---

+ 替代位域的优选方案——EnumSet

  ​	需要传递多组常量集时，有些人回倾向于用位域代替枚举。

  ​	**`java.util.EnumSet`类能有效地表示从单个枚举类型中提取的多个值的多个集合，是位域的优选替代方案**。这个类实现了Set接口，提供了丰富功能和类型安全性，以及可以从其他Set实现中得到的互用性。

  + 在内部实现中，每个EnumSet内容都表示为位矢量。**如果底层的枚举类型有64个或者更少的元素（大多如此）整个EnumSet就使用单个long表示，因此它的性能比得上位域的性能**。
  + **批处理操作，如removeAll和retainAll，都是利用位算法来实现的，就像手工替代位域实现的那样**。但是可以避免手工操作时容易出现的错误以及丑陋的代码，因为EnumSet替你完成了这项艰巨的工作。

  ​	以下ø使用枚举Enum替代前面的位域代码，更加简洁、安全：

  ```java
  // EnumSet - a modern replacement for bit fields
  public class Text {
    public enum Style { BOLD, ITALIC, UNDELINE, STRIKETHROUGH }
    
    // Any Set could be passed in, but EnumSet is clearly best 
    public void applyStyles(Set<Style> styles) { ... }
  }
  ```

  ​	下面是将EnumSet实例传递给applyStyles方法的客户端代码。EnumSet提供了丰富的静态工厂来轻松创建集合，其中一个如下代码所示：

  ```java
  text.applyStyles(EnumSet.of(Style.BOLD, Style.ITALIC));
  ```

  > 注意，applyStyles方法采用的是`Set<Sty1e>`而非`EnumSet<Sty1e>`。虽然看起来好像所有的客户端都可以将EnumSet传到这个方法，但是**最好还是接受接口类型而非接受实现类型**（E64）。这是考虑到可能会有特殊的客户端需要传递一些其他的Set实现。

---

+ 小结

  ​	**正是因为枚举类型要用在集合中，所以没有理由用位域来表示它**。

  + EnumSet比起位域更简洁，且带有（E34）中枚举类型的所有优点。
  + EnumSet缺点，**截止Java9发行版本，它都无法创建不可变的EnumSet**。可以用`Collections.unmodifiableSet将EnumSet`封装起来，但是简洁性和性能会受到影响。

> [EnumSet (Java SE 11 & JDK 11 ) (runoob.com)](https://www.runoob.com/manual/jdk11api/java.base/java/util/EnumSet.html)

## * E37 用EnumMap代替序数索引

+ 概述

  ​	有一种快速的Map实现专门用于枚举键，称作`java.util.EnumMap`。

  ​	**EnumMap构造器采用键类型的Class对象：这是一个有限制的类型令牌（bounded type token），它提供了运行时的泛型信息（E33）**。

---

+ 代码举例

  ​	假设有一个类表示用于烹饪的香草：

  ```java
  class Plant {
    enum LifeCycle { ANNUAL, PERENNIAL, BIENNIAL }
    
    final String name;
    final LifeCycle lifeCycle;
    
    Plant(String name, LifeCycle lifeCycle) {
      this.name = name;
      this.lifeCycle = lifeCycle;
    }
    
    @Override public String toString() {
      return name;
    }
  }
  ```

  ​	假设按照一年生、多年生或者多年生类别对植物进行分类，容易想到需要构建三个集合，每种类型一个，遍历整座花园，将每种香草放到相应的集合中。

  ​	有些程序员会讲这些集合放到一个按照<u>类型的序数</u>进行索引的数组中实现这一点：

  ```java
  // Using ordinal() to index into an array - DON't DO THIS!
  Set<Plant>[] plantsByLifeCycle = 
    (Set<Plant>[]) new Set[Plant.LifeCycle.values().length];
  for (int i = 0; i < plantsByLifeCycle.length; i++)
    plantsByLifrCycle[i] = new HashSet<>();
  
  for(Plant p : garden)
    plantsByLifeCycle[p.lifeCycle.ordinal()].add(p);
  
  // Print the results
  for(int i = 0; i < plantsByLifeCycle.length; i++) {
    System.out.printf("%s: %s%n",
                     Plant.LifeCycle.values()[i], plantsByLifeCycle[i]);
  }
  ```

  ​	上述代码虽可行，但是隐藏许多问题：

  + 数组不能与泛型（E28）兼容，程序需要进行未受检的转换，并且不能正确无误地进行编译。
  + 当你访问一个按照枚举的序数进行索引的数组时，使用正确的int值就是你的职责了。
  + int不能提供枚举的类型安全。

  ​	以下使用EnumMap对代码进行改进：

  ```java
  // Using an EnumMap to associate data with an enum
  Map<Plant.LifeCycle, Set<Plant>> plantsByLifeCycle = 
    new EnumMap<>(Plant.LifeCycle.class);
  for (Plant.LifeCycle lc : Plant.LifeCycle.values())
    plantsByLifeCycle.put(lc, new HashSet<>());
  for (Plant p : garden)
    plantsByLifeCycle.get(p.lifeCycle).add(p);
  System.out.println(plantsByLifeCycle);
  ```

  ​	这段程序更简短、更清楚，也更加安全，运行速度方面可以与使用序数的程序相媲美。它没有不安全的转换；不必手工标注这些索引的输出，因为映射键知道如何将自身翻译成可打印字符串的枚举；计算数组索引时也不可能出错。 

  ​	EnumMap在运行速度方面之所以能与通过序数索引的数组相媲美，正是**因为EnuMap在内部使用了这种数组。但是它对程序员隐藏了这种实现细节，集Map的丰富功能和类型安全与数组的快速于一身**。注意 Enummap构造器采用键类型的 Class对象：这是一个有限制的类型令牌(bounded type token)，它提供了运行时的泛型信息（E33）。

  ​	上一段程序可能比用 stream（E45）管理映射要简短得多。下面是基于 stream的最简单的代码，大量复制了上一个示例的行为：

  ```java
  // Naive stream-based approach - unlikely to produce an EnumMap!
  System.out.println(Arrays.stream(garden)
                     .collect(gropingBy(p -> p.lifeCycle)));
  ```

  ​	这段代码的问题在于它选择自己的映射实现，实际上不会是一个EnumMap，因此<u>与显式EnumMap版本的空间及时间性能并不吻合</u>。为了解决这个问题，要使用有三种参数形式的`Collectors.groupingBy`方法，它允许调用者利用mapFactory参数定义映射实现：

  ```java
  // Using a stream and an EnumMap to associate data with an enum
  System.out.println(Arrays.stream(garden))
    .collect(groupingBy(p -> p.lifeCycle,
                        () -> new EnumMap<>(LifeCycle.class),toSet()));
  ```

  ​	<u>在这样一个玩具程序中不值得进行这种优化，但是在大量使用映射的程序中就很重要了</u>。

  ​	基于 stream的代码版本的行为与EnumMap版本的稍有不同。 EnumMap版本总是给每一个植物生命周期都设计一个嵌套映射，基于 stream的版本则仅当花园中包含了一种或多种植物带有该生命周期时才会设计一个嵌套映射。因此，假如花园中包含了一年生和多年生植物，但没有两年生植物，plantByLifeCycle的数量在 EnumMap版本中应该是三种，在基于 stream的两个版本中则都是两种。

  ​	你还可能见到按照序数进行索引（两次）的数组的数组，该序数表示两个枚举值的映射。例如，下面这个程序就是使用这样一个数组将两个阶段映射到一个阶段过渡中（从液体到固体称作凝固，从液体到气体称作沸腾，诸如此类）：

  ```java
  // Using ordinal() to index array of arrays - DON'T DO THIS!
  public enum Phase {
    SOLID, LIQUID, GAS;
    public enum Transition {
      MELT, FREEZE, BOIL, CONDENSE, SUBLIME, DEPOSIT;
      
      // Rows indexed by from-ordinal, cols by to-ordinal
      private static final Transition[][] TRANSITIONS = {
        {null, MELT, SUBLIME},
        {FREEZE, null, BOIL},
        {DEPOSIT, CONDENSE, null}
      };
      
      // Returns the phase transition from one phase to another
      public static Transition from(Phase from, Phase to) {
        return TRANSITIONS[from.ordinal()][to.ordinal()];
      }
    }
  }
  ```

  ​	这段程序可行，看起来也比较优雅，但是事实并非如此。就像上面那个比较简单的香草花园的示例一样，编译器无法知道序数和数组索引之间的关系。如果在过渡表中出了错，或者在修改`Phase`或者`Phase.Transition`枚举类型的时候忘记将它更新，程序就会在运行时失败。这种失败的形式可能为 ArrayIndexOutOfBoundsException、NullPointerException或者(更糟糕的是)没有任何提示的错误行为。这张表的大小是阶段数的平方，即使非空项的数量比较少。

  ​	同样，利用 EnumMap依然可以做得更好一些。因为每个阶段过渡都是通过一对阶段枚举进行索引的，最好将这种关系表示为一个映射，这个映射的键是一个枚举(起始阶段)，值为另一个映射，这第二个映射的键为第二个枚举(目标阶段，它的值为结果(阶段过渡，即形成了Map(起始阶段，Map(目标阶段，阶段过渡)这种形式。一个阶段过渡所关联的两个阶段，最好通过"数据与阶段过渡枚举之间的关系"获取，之后用该阶段过渡枚举来初始化嵌套的EnumMap：

  ```java
  // Using a nested EnumMap to associate data with enum pairs
  public enum Phase {
    SOLID, LIQUID, GAS;
    public enum Transition {
      MELT(SOLID, LIQUID), FREEZE(LIQUID, SILID),
      BOIL(LIQUID, GAS), CONDENSE(GAS, LIQUID),
      SUBLIME(SOLID, GAS), DEPOSIT(GAS, SOLID);
  
      private final Phase from;
      private final Phase to;
  
      Transition(Phase from, Phase to) {
        this.from = from;
        this.to = to;
      }
  
      // Initialize the phase transition map
      private static final Map<Phase, Map<Phase, Transition>>
        m = Stream.of(values()).collect(gropingBy(t -> t.from, () -> new EnumMap<>(Phase.class), toMap(t -> t.to, t -> t, (x, y) -> y, () -> new EnumMap<>(Phase.class))));
  		
      public static Transition from(Phase from, Phase to) {
        return m.get(from).get(to);
      }
    }
  }
  ```

  ​	初始化阶段过渡映射的代码看起来可能有点复杂。映射的类型为`Map<Phase，Map<Phase， Transition>>`，表示是由键为源Phase(即第一个阶段)、值为另一个映射组成的Map，其中组成值的Map是由键值对目标Phase(即第二个阶段)和Transition组成的。这个映射的映射是利用两个集合的级联顺序进行初始化的。第一个集合按源Phase对过渡进行分组，第二个集合利用从目标Phase到过渡之间的映射创建一个 EnumMap。第个集合中的merge函数`((x，y)->y)`没有用到；只有当我们因为想要获得一个 EnumMap而定义映射工厂时才需要用到它，同时Collectors提供了重叠工厂。本书第2版是利用显式迭代来初始化阶段过渡映射的。其代码更加烦琐，但是的确更易于理解。

  ​	现在假设想要给系统添加一个新的阶段：plasma(离子)或者电离气体。只有两个过渡与这个阶段关联：电离化(ionization)，它将气体变成离子；以及消电离化(deionization)，将离子变成气体。为了更新基于数组的程序，必须给Phase添加一种新常量，给`Phase.Transition`添加两种新常量，用一种新的16个元素的版本取代原来9个元素的数组的数组。如果给数组添加的元素过多或者过少，或者元素放置不妥当，可就麻烦了：程序可以编译，但是会在运行时失败。为了更新基于 EnumMap的版本，所要做的就是必须将PLASMA添加到 Phase列表，并将工IONIZE(GAS， PLASMA)和DEIONIZE(PLASMA，GAS)添加到`Phase.Transition`的列表中：

  ```java
  // Adding a new phase using the nested EnumMap implementation
  public enum Phase {
    SOLID, LIQUID, GAS, PLASMA;
    public enum Transition {
      MELT(SOLD, LIQUID), FREEZR(LIQUID, SOLID),
      BOIL(LIQUID, GAS), CONDENSE(GAS, LIQUID),
      SUBLIME(SOLID, GAS), DEPOSIT(GAS, SOLID),
      IONIZE(GAS, PLASMA), DEIONIZE(PLASMA, GAS);
      ...// Remainder unchanged
    }
  }
  ```

  ​	程序会自行处理所有其他的事情，这样就几乎没有出错的可能。从内部来看，映射的映射被实现成了数组的数组，因此在提升了清晰性、安全性和易维护性的同时，在空间或者时间上也几乎没有多余的开销。

  ​	为了简洁起见，上述范例是用null表明状态没有变化(这里的to和from是相等的)。这并不是好的实践，可能在运行时导致NullPointerException异常。要给这个问题设计个整洁、优雅的解决方案，需要高超的技巧，得到的程序会很长，贬损了本条目的主要精神。

---

+ 小结

  ​	总而言之，**最好不要用序数来索引数组，而要使用 EnumMap**。

  ​	如果你所表示的这种关系是多维的，就使用`EnumMap<... , EnumMap<...>>`。应用程序的程序员在一般情况下都不使用`Enum.ordinal`方法，仅仅在极少数情况下才会使用，因此这是一种特殊情况（E35）。

## * E38 用接口模拟可拓展的枚举

+ 概述

  ​	从多方面来看，枚举类型优于本书第1版中描述的类型安全枚举模式。第1版所述的模式能实现让一个枚举类型去扩展另一个枚举类型；利用这种语言特性，则不可能做到。

  ​	**枚举的可伸缩性最后证明基本上都不是什么好点子**。

  ​	目前还没有很好的方法来枚举基本类型的所有元素及其扩展。最终，可伸缩性会导致设计和实现的许多方面变得复杂起来。

  ​	<u>对于可伸缩的枚举类型而言，至少有一种具有说服力的用例，这就是操作码(operation code)，也称作 opcode</u>。操作码是指这样的枚举类型：它的元素表示在某种机器上的那些操作，例如（E34）中的 Operation类型，它表示一个简单的计算器中的某些函数。有时要尽可能地让API的用户提供它们自己的操作，这样可以有效地扩展API所提供的操作集。

  ​	幸运的是，有一种很好的方法可以利用枚举类型来实现这种效果。由于枚举类型可以通过给操作码类型和（属于接口的标准实现的)枚举定义接口来实现任意接口，基本的想法就是利用这一事实。例如，以下是（E34）中的Operation类型的扩展版本：

  ```java
  public interface Operation {
    double apply(double x, double y);
  }
  
  public enum BasicOperation implements Operation {
    PLUS("+") {
      public double apply(double x, double y) { return x + y; }
    },
    MINUS("-") {
      public double apply(double x, double y) { return x - y; }
    },
    TIMES("*") {
      public double apply(double x, double y) { return x * y; }
    },
    DIVIDE("/") {
      public double apply(double x, double y) { return x / y; }
    };
    private final String symbol;
    
    BasicOepration(String symbol) {
      this.symbol = symbol;
    }
    
    @Override
    public String toString() {
      return symbol;
    }
  }
  ```

  ​	在可以使用基础操作的任何地方，现在都可以使用新的操作，只要API是写成采用接类型(Operation)而非实现(BasicOperation)。注意，在枚举中，不必像在不可扩展的枚举中所做的那样，利用特定于实例的方法实现（E34）来声明抽象的apply方法。因为抽象的方法(apply)是接口(Operation)的一部分。

  ​	不仅可以在任何需要"基本枚举"的地方单独传递一个"扩展枚举"的实例，而且除了那些基本类型的元素之外，还可以传递完整的扩展枚举类型，并使用它的元素。例如，通过(E34)的测试程序版本，体验一下上面定义过的所有扩展过的操作：

  ```java
  public static void main(String[] args) {
    double x = Double.parseDounle(args[0]);
    double y = Double.parseDouble(args[1]);
    test(ExtendedOperation.class, x, y);
  }
  
  private static <T extends Enum<T> & Operation> void test(
  Class<T> opEnumType, double x, double y) {
    for(Operation op : opEnumType.getEnumConstants())
      System.out.printf("%f %s %f = %f%n", x, op, y, op.apply(x,y));
  }
  ```

  ​	注意扩展过的操作类型的类的字面文字（ExtendedOperation.class）从main被传递给了test方法，来描述被扩展操作的集合。这个类的字面文字充当有限制的类型令牌(bounded type token)（E33）。opEnumType**参数中公认很复杂的声明(`<T extends Enum<T> & operation> C1ass<T>`)确保了Class对象既表示枚举又表示Operation的子类型，这正是遍历元素和执行与每个元素相关联的操作时所需要的**。

  ​	第二种方法是传入一个`Collection<? Extends Operation>`，这是个有限制的通配符类型(bounded wildcard type)（E31），而不是传递一个类对象：

  ```java
  public static void main(String[] args) {
    double x = Double.parseDouble(args[0]);
  	double y = Double.parseDouble(args[1]);
    test(Arrays.asList(ExtendedOperation.values()), x, y);
  }
  
  private static void test(Collection<? extends Operation> opSet, double x, double y) {
    for (Operation op : opSet)
      System.out.printf("%f %s %f = %f%n", x, op, y, op.apply(x, y));
  }
  ```

  ​	这样得到的代码没有那么复杂，test方法也比较灵活一些：<u>它允许调用者将多个实现类型的操作合并到一起。另一方面，也放弃了在指定操作上使用 EnumSet（E36）和 EnumMap（E37）的功能</u>。

  ​	上面这两段程序运行时带上命令行参数4和2，都会产生如下输出：

  ```shell
  4.000000 ^ 2.000000 = 16.000000
  4.000000 % 2.000000 = 0.000000
  ```

  ​	<u>用接口模拟可伸缩枚举有个小小的不足，即无法将实现从一个枚举类型继承到另一个枚举类型</u>。如果实现代码不依赖于任何状态，就可以将缺省实现（E20）放在接口中。在上述 Operation的示例中，保存和获取与某项操作相关联的符号的逻辑代码，必须复制到BasicOperation和 ExtendedOperation中。在这个例子中是可以的，因为复制的代码非常少。如果共享功能比较多，则可以将它封装在一个辅助类或者静态辅助方法中来避免代码的复制工作。

  ​	本条目所述的模式也在Java类库中得到了应用。例如，`java.nio.file.LinkOption`枚举类型，它同时实现了CopyOption和 OpenOption接口。

---

+ 小结

  ​	总而言之，**虽然无法编写可扩展的枚举类型，却可以通过编写接口以及实现该接口的基础枚举类型来对它进行模拟**。这样允许客户端编写自己的枚举(或者其他类型)来实现接口。如果API是根据接口编写的，那么在可以使用基础枚举类型的任何地方，也都可以使用这些枚举。

## E39 注解优先于命名模式

+ 概述

  ​	**根据经验，一般使用命名模式（naming pattern）表明有些程序元素需要通过某种工具或者框架进行特殊处理**。

  ​	例如，在Java4发行版本之前，Jnit测试框架原本要求其用户定要用test作为测试方法名称的开头[Beck04]。**这种方法可行，但是有几个很严重的缺点**。

  + <u>文字拼写错误会导致失败，且没有任何提示</u>。

    例如，假设不小心将一个测试方法命名为 tsetSafetyOverride而不是 testSafetyOverride。 Junit3不会提示，但也不会执行测试，造成错误的安全感。

  + <u>无法确保它们只用于相应的程序元素上</u>。

    例如，假设将某个类称作 TestSafetyMechanisms，是希望JUnt3会自动地测试它所有的方法，而不管它们叫什么名称。 Junit3还是不会提示，但也同样不会执行测试。

  + <u>它们没有提供将参数值与程序元素关联起来的好方法</u>。

    例如，假设想要支持一种测试类别，它只在抛出特殊异常时才会成功。异常类型本质上是测试的一个参数。你可以利用某种具体的命名模式，将异常类型名称编码到测试方法名称中，但是这样的代码很不雅观，也很脆弱（E62）。编译器不知道要去检验准备命名异常的字符串是否真正命名成功。如果命名的类不存在，或者不是一个异常，你也要到试着运行测试时才会发现。

  ​	注解[JLS，9.7]很好地解决了所有这些问题，Jnit从Java4开始使用。<u>在本条目中，我们要编写自己的试验测试框架，展示一下注解的使用方法</u>。假设想要定义一个注解类型来指定简单的测试，它们自动运行，并在抛出异常时失败。以下就是这样的一个注解类型，命名为Test：

  ```java
  // Marker annotation type declaration
  import java.lang.annotation.*;
  /**
   * Indicates that the annotated method is a test method.
   * Use only on parameterless static methods.
   */
  @Retention(RetentionPolicy.RUNTIME)
  @Target(ElememtType.METHOD)
  public @interface Test {
  }
  ```

  ​	Test注解类型的声明就是它自身通过@Retention和@Target注解进行了注解。

  ​	**注解类型声明中的这种注解被称作元注解(meta-annotation)**。

  + `@Retention(RetentionPolicy.RUNTIME)`元注解表明Test注解在运行时也应该存在，否则测试工具就无法知道Test注解。
  + `@Target(ElementType. METHOD)`元注解表明，Test注解只在方法声明中才是合法的：它不能运用到类声明、域声明或者其他程序元素上。

  ​	注意Test注解声明上方的注释："Use only on parameterless static method"(只用于无参的静态方法)。如果编译器能够强制这一限制最好，但是它做不到，除非编写一个**注解处理器(annotation processor)**，让它来完成。关于这个主题的更多信息，请参阅 Javax.annotation.processing的文档。在没有这类注解处理器的情况下，如果将Test注解放在实例方法的声明中，或者放在带有一个或者多个参数的方法中，测试程序还是可以编译，让测试工具在运行时来处理这个问题。

  ​	下面就是现实应用中的Test注解，称作**标记注解(marker annotation)**，因为它没有参数，只是"标注"被注解的元素。如果程序员拼错了Test，或者将Test注解应用到程序元素而非方法声明，程序就无法编译：

  ```java
  // Program containing marker annotations
  public class Sample {
    @Test
    public static void m1() { } // Test should pass
    public static void m2() { }
    @Test
    public static void m3() { // Test should fail
      throw new RuntimeException("Boom");
    }
    public static void m4() { }
    @Test
    public void m5() { } // INVALID USE: nonstatic method
    public static void m6() { }
    @Test
    public static void m7() { // Test should fail
      throw new RuntimeException("Cash");
    }
    public static void m8() { }
  }
  ```

  ​	Sample类有7个静态方法，其中4个被注解为测试。这4个中有2个抛出了异常m3和m7，另外两个则没有：m1和m5。但是其中一个没有抛出异常的被注解方法：m5，是个实例方法，因此不属于注解的有效使用。总之，Sample包含4项测试：一项会通过，两项会失败，另一项无效。没有用Test注解进行标注的另外4个方法会被测试工具忽略。

  ​	Test注解对 Sample类的语义没有直接的影响。它们只负责提供信息供相关的程序使用。更一般地讲，注解永远不会改变被注解代码的语义，但是使它可以通过工具进行特殊的处理，例如像这种简单的测试运行类：

  ```java
  // Program to process marker annotations
  import java.lang.reflect.*;
  
  public class RunTests {
    public static void main(String[] args) throws Exception {
      int tests = 0;
      int passed = 0;
      Class<?> testClass = Class.forName(args[0]);
      for (Method m : testClass.getDeclaredMethods()) {
        if(m.isAnnotationPresent(Test.class)) {
          tests++;
          try {
            m.invoke(null);
            passed++;
          } catch (InvocationTargetException wrappedExc) {
            Throwable exc = wrappedExc.getCause();
            System.out.println(m + " failed: " + exc);
          } catch (Exception exc) {
            System.out.println("Invalid @Test: " + m);
          }
        }
      }
      System.out.printf("Passed: %d, Failed: %d%n", passed, tests - passed);
    }
  }
  ```

  ​	测试运行工具在命令行上使用完全匹配的类名，并通过调用 Method.invoke反射式地运行类中所有标注了Test注解的方法。 isAnnotationPresent方法告知该工具要运行哪些方法。如果测试方法抛出异常，反射机制就会将它封装在InvocationtTargetException中。该工具捕捉到这个异常，并打印失败报告，包含测试方法抛出的原始异常，这些信息是通过getCause方法从InvocationTargetException中提取出来的。

  ​	<u>如果尝试通过反射调用测试方法时抛出InvocationTargetException之外的任何异常，表明编译时没有捕捉到Test注解的无效用法。这种用法包括实例方法的注解，或者带有一个或多个参数的方法的注解，或者不可访问的方法的注解。测试运行类中的第二个catch块捕捉到这些Test用法错误，并打印出相应的错误消息</u>。下面就是 RunTests在Sample上运行时打印的输出:

  ```shell
  public static void Sample.m3() failed: RuntimeException: Boom Invalid @Test: public void Sample.m5()
  public static void Sample.m7() failed: RuntimeException: Crash Passed : 1, Failed: 3
  ```

  ​	现在我们要针对只在抛出特殊异常时才成功的测试添加支持。为此需要一个新的注解类型：

  ```java
  // Annotation type with a parameter
  import java.lang.annotation.*;
  /**
  * Indicates that the annotated method is a test method that must throw the designated exception to succeed.
  */
  @Retention(RetentionPolicy.RUNTIME)
  @Target(ElementType.METHOD)
  public @interface ExceptionTest {
    Class<? extends Throwable> value();
  }
  ```

  ​	<u>这个注解的参数类型是`Class<? extends Throwable>`。这个通配符类型有些绕口。它在英语中的意思是：某个扩展 Throwable的类的Class对象，它允许注解的用户指定任何异常(或错误)类型</u>。这种用法是有限制的类型令牌(bounded type token)（E33）的一个示例。下面就是实际应用中的这个注解。注意类名称被用作了注解参数的值：

  ```java
  // Program containing annotations with a parameter
  public class Sample2 {
    @ExceptionTest(ArithmeticException.class)
    public static void m1() { // Test should pass
      int i = 0;
      i = i / i;
    }
    @ExceptionTest(ArithmeticException.class)
    public static void m2() { // Should fail (wrong exception)
      int[] a = new int[0];
      int i = a[i]; 
    }
    @ExceptionTest(ArithmeticException.class)
    public static void m3() { } // Should fail(no exception)
  }
  ```

  ​	现在我们要修改一下测试运行工具来处理新的注解。这其中包括将以下代码添加到main方法中：

  ```java
  if (m.isAnnotationPresent(ExceptionTest.class)) {
    tests++;
    try {
      m.invoke(null);
      System.out.printf("Test %s failed: no exception%n", m);
    } catch (InvocationTargetException wrappedEx) {
      Class<? extends Throwable> excType = 
        m.getAnnotation(ExceptionTest.class).value();
      if(exType.isInstance(exc)) {
        passed++;
      } else {
        System.out.printf("Test %s failed: expected %s, got %s%n", m, excType.getName(), exc);
      }
    } catch (Exception exc) {
      System.out.println("Invalid @Test: " + m);
    }
  }
  ```

  ​	这段代码类似于用来处理Test注解的代码，但有一处不同：这段代码提取了注解参数的值，并用它检验该测试抛出的异常是否为正确的类型。没有显式的转换，因此没有出现ClassCastException的危险。编译过的测试程序确保它的注解参数表示的是有效的异常类型，<u>需要提醒一点：有可能注解参数在编译时是有效的，但是表示特定异常类型的类文件在运行时却不存在。在这种希望很少出现的情况下，测试运行类会抛出 TypeNotPresentException异常</u>。

  ​	将上面的异常测试示例再深入一点，想象测试可以在抛出任何一种指定异常时都能够通过。注解机制有一种工具，使得支持这种用法变得十分容易。假设我们将ExceptionTest注解的参数类型改成Class对象的一个数组：

  ```java
  // Annotation type with an array parameter
  @Retention(RetentionPolicy.RUNTIME)
  @Target(ElementType.METHOD)
  public @interface ExceptionTest {
    Class<? extends Exception>[] value();
  }
  ```

  ​	注解中数组参数的语法十分灵活。它是进行过优化的单元素数组。使用了ExceptionTest新版的数组参数之后，<u>之前的所有 ExceptionTest注解仍然有效，并产生单元素的数组</u>。为了指定多元素的数组，要用花括号将元素包围起来，并用逗号将它们隔开：

  ```java
  // Code containing an annotation with an array parameter
  @ExceptionTest({ IndexOutOfBoundsException.class, NullPointerException.class })
  public static void doublyBad() {
    List<String> list = new ArrayList<>();
    
    // The spec permits this method to throw either
    // IndexOutOfBoundsException or NullPointerException
    list.addAll(5, null);
  }
  ```

  ​	修改测试运行工具来处理新的ExceptionTest相当简单。下面的代码代替了原本的代码：

  ```java
  if (m.isAnnotationPresent(ExceptionTest.class)) {
    tests++;
    try{
      m.invoke(null);
      System.out.printf("Test %s failed: no exception%n", m);
    } catch (Throwable wrappedExc) {
      Throwable exc = wrappedExc.getCause();
      int oldPassed = passed;
      Class<? extends Exception>[] excTypes = m.getAnnotation(ExceptionTest.class).value();
      for (Class<? extends Exception> excType : excTypes) {
        if (excType.isInstance(exc)) {
          passed++;
          break;
        }
      }
      if (passed == oldPassed)
        System.out.printf("Test %s failed: %s %s", m, exc);
    }
  }
  ```

  ​	<u>从Java8开始，还有另一种方法可以进行多值注解。它不是用一个数组参数声明一个注解类型，而是用@Repeatable元注解对注解的声明进行注解，表示该注解可以被重复地应用给单个元素。这个元注解只有一个参数，就是包含注解类型(containing annotation type)的类对象，它唯一的参数是一个注解类型数组[JLS，9.6.3]</u>。下面的注解声明就是把ExceptionTest注解改成使用这个方法之后的版本。注意包含的注解类型必须利用适当的保留策略和目标进行注解，否则声明将无法编译：

  ```java
  // Repeatable annotation type
  @Retention(RetentionPolicy.RUNTIME)
  @Target(ElementType.METHOD)
  @Repeatable(ExceptionTestContainer.class)
  public @interface ExceptionTest {
    Class<? extends Exception> value();
  }
  
  @Retention(RetentionPolicy.RUNTIME)
  @Target(ElementType.METHOD)
  public @interface ExceptionTestContainer {
    ExceptionTest[] value();
  }
  ```

  ​	下面是doublyBad测试方法用重复注解代替数组值注解之后的代码：

  ```java
  // Code containing a repeated annotation
  @RetentionTest(IndexOutOfBoundException.class)
  @RetentionTest(NullPonterException.class)
  public static void doublyBad() { ... }
  ```

  ​	**处理可重复的注解要非常小心。重复的注解会产生一个包含注解类型的合成注解**。

  ​	<u>getAnnotationsByType方法掩盖了这个事实，可以用于访问可重复注解类型的重复和非重复的注解</u>。<u>但isAnnotationPresent使它变成了显式的，即重复的注解不是注解类型(而是所包含的注解类型)的一部分</u>。

  ​	<u>如果一个元素具有某种类型的重复注解，并且用isAnnotationPresent方法检验该元素是否具有该类型的注解，会发现它没有。用这种方法检验是否存在注解类型，会导致程序默默地忽略掉重复的注解。同样地，用这种方法检验是否存在包含的注解类型，会导致程序默默地忽略掉非重复的注解</u>。

  ​	为了利用isAnnotationPresent检测重复和非重复的注解，必须检查注解类型及其包含的注解类型。下面是 Runtests程序改成使用ExceptionTest注解时有关部分的代码：

  ```java
  // Processing repeatable annotations
  if (m.isAnnotationPresent(ExceptionTest.class)
     || m.isAnnotationPresent(ExceptionTestContainer.class)) {
    tests++;
    try {
      m.invoke(null);
      System.out.printf("Test %s failed: no exception%n", m);
    } catch (Throwable wrappedExc) {
      Throwable exc = wrappedExc.getCause();
      int oldPassed = passed;
      ExceptionTest[] excTests = m.getAnnotationsByType(ExceptionTest.class);
      for (ExceptionTest excTest : excTests) {
        if  excTest.value().isInstance(exc)) {
          passed++;
          break;
        }
      }
      if(passed == oldPassed)
        System.out.printf("Test %s failed: %s %n", m, exc);
    }
  }
  ```

  ​	加入可重复的注解，提升了源代码的可读性，逻辑上是将同一个注解类型的多个实例应用到了一个指定的程序元素。如果你觉得它们增强了源代码的可读性就使用它们，但是记住<u>在声明和处理可重复注解的代码中会有更多的样板代码，并且处理可重复的注解容易出错</u>。

  ​	本条目中的测试框架只是一个试验，但它清楚地示范了注解之于命名模式的优越性这只是揭开了注解功能的冰山一角。如果是在编写一个需要程序员给源文件添加信息的工具，就要定义一组适当的注解类型。**既然有了注解，就完全没有理由再使用命名模式了**。

  ​	也就是说，除了"工具铁匠"( toolsmiths，即平台框架程序员)之外，大多数程序员都不必定义注解类型。但是**所有的程序员都应该使用Java平台所提供的预定义的注解类型**（E40和E27）。还要考虑使用IDE或者静态分析工具所提供的任何注解。这种注解可以提升由这些工具所提供的诊断信息的质量。但是要注意这些注解还没有标准化，因此如果变换工具或者形成标准，就有很多工作要做了。

> [Java注解处理之反射API_liuxigiant的专栏-CSDN博客](https://blog.csdn.net/liuxigiant/article/details/54296275)
>
> [深度学习Java注解 - SegmentFault 思否](https://segmentfault.com/a/1190000022861141)
>
> [为什么Java注解元素不能是包装类? - 知乎 (zhihu.com)](https://www.zhihu.com/question/342626971)

---

+ 自己的简易代码测试

  ​	确实平时写注解的场景很少，或者说就算用到，也是很久写一次，不至于每次开发某些东西都需要写，所以注解代码开发，比较容易生疏。

  1. TestAnnotation注解类

     ```java
     package annotation;
     
     import java.lang.annotation.*;
     
     // 下面这两行加不加，3. 测试代码的输出都是一样的
     // @Retention(RetentionPolicy.RUNTIME)
     // @Target(ElementType.TYPE)
     @Repeatable(AnnotationTestContainer.class)
     public @interface TestAnnotation {
       // 这里用不了Double ，编译器会报错
         double[] value() default {0,0.1};
     }
     ```

  2. AnnotationTestContainer注解容器类

     ```java
     package annotation;
     
     import java.lang.annotation.ElementType;
     import java.lang.annotation.Retention;
     import java.lang.annotation.RetentionPolicy;
     import java.lang.annotation.Target;
     
     @Retention(RetentionPolicy.RUNTIME)
     @Target(ElementType.TYPE)
     public @interface AnnotationTestContainer {
         TestAnnotation[] value();
     }
     ```

  3. 测试代码

     ```java
     package annotation;
     
     import java.lang.annotation.Annotation;
     
     public class AnnotationTest {
       @TestAnnotation(value = {4.0,3.2})
       @TestAnnotation(value = {4.0})
       @TestAnnotation(value = {5.0})
       @TestAnnotation
       @TestAnnotation
       static class Test001 {
     
       }
     
       public static void main(String[] args) {
         Test001 test001 = new Test001();
         Annotation[] annotations = test001.getClass().getAnnotations();
         // 1
         System.out.println(annotations.length);
         // @annotation.AnnotationTestContainer(value=[@annotation.TestAnnotation(value=[4.0, 3.2]), @annotation.TestAnnotation(value=[4.0]), @annotation.TestAnnotation(value=[5.0]), @annotation.TestAnnotation(value=[0.0, 0.1]), @annotation.TestAnnotation(value=[0.0, 0.1])])
         for(Annotation annotation : annotations){
           System.out.println(annotation.toString());
         }
         Class<Test001> test001Class = Test001.class;
         TestAnnotation[] annotationsByType = test001Class.getAnnotationsByType(TestAnnotation.class);
         // @annotation.TestAnnotation(value=[4.0, 3.2])
         // @annotation.TestAnnotation(value=[4.0])
         // @annotation.TestAnnotation(value=[5.0])
         // @annotation.TestAnnotation(value=[0.0, 0.1])
         // @annotation.TestAnnotation(value=[0.0, 0.1])
         for(TestAnnotation testAnnotation : annotationsByType) {
           System.out.println(testAnnotation.toString());
         }
         AnnotationTestContainer[] annotationsByType1 = test001Class.getAnnotationsByType(AnnotationTestContainer.class);
         // @annotation.AnnotationTestContainer(value=[@annotation.TestAnnotation(value=[4.0, 3.2]), @annotation.TestAnnotation(value=[4.0]), @annotation.TestAnnotation(value=[5.0]), @annotation.TestAnnotation(value=[0.0, 0.1]), @annotation.TestAnnotation(value=[0.0, 0.1])])
         for(AnnotationTestContainer annotationTestContainer: annotationsByType1) {
           System.out.println(annotationTestContainer);
         }
       }
     }
     ```

  4. 输出如下：

     ```java
     1
     @annotation.AnnotationTestContainer(value=[@annotation.TestAnnotation(value=[4.0, 3.2]), @annotation.TestAnnotation(value=[4.0]), @annotation.TestAnnotation(value=[5.0]), @annotation.TestAnnotation(value=[0.0, 0.1]), @annotation.TestAnnotation(value=[0.0, 0.1])])
     @annotation.TestAnnotation(value=[4.0, 3.2])
     @annotation.TestAnnotation(value=[4.0])
     @annotation.TestAnnotation(value=[5.0])
     @annotation.TestAnnotation(value=[0.0, 0.1])
     @annotation.TestAnnotation(value=[0.0, 0.1])
     @annotation.AnnotationTestContainer(value=[@annotation.TestAnnotation(value=[4.0, 3.2]), @annotation.TestAnnotation(value=[4.0]), @annotation.TestAnnotation(value=[5.0]), @annotation.TestAnnotation(value=[0.0, 0.1]), @annotation.TestAnnotation(value=[0.0, 0.1])])
     ```

## * E40 坚持使用Override注解

+ 概述

  ​	Java类库中包含了几种注解类型。对于传统的程序员而言，其中最重要的就是**@Override注解**。该注解只能用在方法声明中，表示被注解的方法声明覆盖了超类型中的一个方法声明。如果坚持使用这个注解，可以防止一大类的非法错误。

  ​	使用@Override注解，可以让编译器帮助发现重写错误（即自以为重写了，实际是重载的这种错误）。

---

+ 代码举例

  ​	例，这里的Bigram类表示一个双字母组或者有序的字母对：

  ```java
  // Can you spot the bug?
  public class Bigram {
    private final char first;
    private final char second;
    
    public Bigram(char first, char second) {
      this.first = first;
      this.second = second;
    }
    public boolean equals(Bigram b) {
      return b.first == first && b.second == second;
    }
    public int hashCode() {
      return 31 * first + second;
    }
    
    public static void main(String[] args) {
      Set<Bigram> s = new HashSet<>();
      for(int i = 0; i < 10; i++)
        for(char ch = 'a'; ch <= 'z'; ch++)
          s.add(new Bigram(ch, ch));
      System.out.println(s.size());
    }
  }	
  ```

  ​	理想情况下，上述程序将输出26，因为Set里的元素不能重复。实际上，会发现打印260。

  ​	上述代码显然想覆盖equals方法（E10）、hashCode方法（第11章），但实际是重载了equals方法（E52）。

  ​	为了覆盖`Object.equals`，必须定义一个参数为Object类型的equals方法，但是<u>上述代码equals方法的参数并非Object类型，因此Bigram类从Object继承了equals方法。这个euqals方法测试对象的同一性（identity），就\=\=操作符一样</u>。每个bigram的10个备份中，每一个都与其余的9个不同（new的原因），因此Object.equals认为它们不相等，导致程序打印260。

  ​	通过@Override标注`Bigram.equals`，告知编译器我们想覆盖`Object.equals`方法：

  ```java
  @Override
  public boolean equals(Bigram b) {
    return b.first == first && b.second == second;
  }
  ```

  ​	如果插入这个注解，尝试重新编译程序，编译器就会产生如下错误信息：

  ```shell
  Bigram.java:10: method does not override or implement a method from a supertype
    @Override public boolean equals(Bigram b) {
    ^
  ```

  ​	此时就会立刻意识到哪里出错了，重新编写正确的equals实现（E10）：

  ```java
  @Override public boolean equals(Object o) {
    if (!(o instanceof Bigram))
      return false;
    Bigram b = (Bigram) o;
    return b.first == first && b.second == second;
  }
  ```

  ​	因此，**应该在你想要覆盖超类声明的每个方法声明中使用 Override注解**。这一规则有个小小的例外。如果你在编写一个没有标注为抽象的类，并且确信它覆盖了超类的抽象方法，在这种情况下，就不必将Override注解放在该方法上了。在没有声明为抽象的类中，如果没有覆盖抽象的超类方法，编译器就会发出一条错误消息。但是，你可能希望关注类中所有覆盖超类方法的方法，在这种情况下，也可以放心地标注这些方法。

  ​	*大多数IDE可以设置为在需要覆盖一个方法时自动插入 Override注解。大多数IDE都提供了使用 Override注解的另一种理由。如果启用相应的代码检验功能，当有一个方法没有 Override注解，却覆盖了超类方法时，IDE就会产生一条警告。如果使用了Override注解，这些警告就会提醒你警惕无意识的覆盖。这些警告补充了编译器的错误消息，后者会提醒你警惕无意识的覆盖失败。IDE和编译器可以确保你无一遗漏地覆盖任何你想要覆盖的方法。*

  ​	*Override注解可以用在方法声明中，覆盖来自接口以及类的声明。由于缺省方法的岀现，在接口方法的具体实现上使用 Override，可以确保签名正确，这是一个很好的实践。如果知道接口没有缺省方法，可以选择省略接口方法的具体实现上的 Override注解，以减少混乱。*

  ​	<u>但是在抽象类或者接口中，还是值得标注所有你想要的方法，来覆盖超类或者超接口方法，无论它们是具体的还是抽象的</u>。例如，Set接口没有给Collection接口添加新方法，因此它应该在它的所有方法声明中包括 Override注解，以确保它不会意外地给Collection接口添加任何新方法。

---

+ 小结

  ​	总而言之，**如果在你想要的每个方法声明中使用Override注解来覆盖超类声明，编译器就可以替你防止大量的错误**，但有一个例外。在具体的类中，不必标注你确信覆盖了抽象方法声明的方法(虽然这么做也没有什么坏处)。

## * E41 用标记接口定义类型

+ 概述

  ​	**标记接口（marker interface）是不包含方法声明的接口，它只是指明（或者"标明"）一个类实现了具有某种属性的接口**。

  ​	例如，考虑 Serializable接口(详见第12章)。通过实现这个接口，类表明它的实例可以被写到 ObjectOutputStream中(或者"被序列化")。

  ​	你可能听说过标记注解（E39）使得标记接口过时了。这种断言是不正确的。标记接口有两点胜过标记注解。首先，也是最重要的一点是，**标记接口定义的类型是由被标记类的实例实现的；标记注解则没有定义这样的类型**。<u>标记接口类型的存在，允许你在编译时就能捕捉到在使用标记注解的情况下要到运行时才能捕捉到的错误</u>。

  ​	<u>Java的序列化设施(详见第6章)利用Serializable标记接口表明一个类型是可以序列化的。 `ObjectOutputStream.writeObject`方法将传入的对象序列化，其参数必须是可序列化的。该方法的参数类型应该为 Serializable，如果试着序列化一个不恰当的对象，（通过类型检査）在编译时就会被发现</u>。<u>编译时的错误侦测是标记接口的目的</u>，但遗憾的是，`ObjectOutputStream.write` API并没有利用Serializable接口的优势：其参数声明为Object类型，因此，如果尝试序列化一个不可序列化的对象，将直到程序运行时才会失败。

  ​	**标记接口胜过标记注解的另一个优点是，它们可以被更加精确地进行锁定**。如果注解类型用目标 ElementType.TYPE声明，它就可以被应用于任何类或者接口。假设有一个标记只适用于特殊接口的实现，如果将它定义成一个标记接口，就可以用它将唯一的接口扩展成它适用的接口，确保所有被标记的类型也都是该唯一接口的子类型。

  ​	Set接口可以说就是这种有限制的标记接口(restricted marker interface)。它只适用于Collection子类，但是它不会添加除了Collection定义之外的方法。一般情况下，不把它当作是标记接口，因为它改进了几个Collection方法的合约，包括add、equals和hashCode。但是很容易想象只适用于某种特殊接口的子类型的标记接口，它没有改进接口的任何方法的合约。<u>这种标记接口可以描述整个对象的某个约束条件，或者表明实例能够利用其他某个类的方法进行处理(就像Serializable接口表明实例可以通过ObjectOutputStream进行处理一样)。</u>

  ​	**标记注解胜过标记接口的最大优点在于，它们是更大的注解机制的一部分**。因此，标记注解在那些支持注解作为编程元素之一的框架中同样具有一致性。

  ​	那么什么时候应该使用标记注解，**什么时候应该使用标记接口呢**？

  + 很显然，如果标记是应用于任何程序元素而不是类或者接口，就必须使用注解，因为<u>只有类和接口可以用来实现或者扩展接口</u>。
  + <u>如果标记只应用于类和接口，就要问问自己：我要编写一个还是多个只接受有这种标记的方法呢？如果是这种情况，就应该优先使用标记接口而非注解</u>。这样你就可以用接口作为相关方法的参数类型，它可以真正为你提供编译时进行类型检查的好处。
  + 如果你确信自己永远不需要编写一个只接受带有标记的对象，那么或许最好使用标记注解。
  + 此外，**如果标记是广泛使用注解的框架的一个组成部分，则显然应该选择标记注解**。

---

+ 小结

  ​	总而言之，标记接口和标记注解都各有用处。

  + 如果想要定义一个任何新方法都不会与之关联的类型，标记接口就是最好的选择。
  + 如果想要标记程序元素而非类和接口，或者标记要适合于已经广泛使用了注解类型的框架，那么标记注解就是正确的选择。
  + **如果你发现自己在编写的是目标为Elementtype.TYPE的标记注解类型，就要花点时间考虑清楚，它是否真的应该为注解类型，想想标记接口是否会更加合适**。

  ​	从某种意义上说，本条目与（E22）中"如果不想定义类型就不要使用接口"的说法相反。本条目最接近的意思是说："如果想要定义类型，一定要使用接口。"

# 7、Lambda和Stream

​	Java8增加了函数接口（functional interface）、Lambda和方法引用（method reference），使得创建函数对象（function objcet）变得容易。

​	与此同时，还增加Stream API，为处理数据元素的序列提供了类库级别的支持。

## * E42 Lambda优先于匿名类

+ 概述

  ​	根据以往的经验，是用带有单个抽象方法的接口（或者，几乎都不是抽象类）作为函数类型（function type）。它们的实例称作函数对象（function object），表示函数或者要采取的动作。

  ​	自从1997年发布JDK1.1以来，创建函数对象的主要方式是通过匿名类（anonymous class，详见E24)。下面是一个按照字符串的长度对字符串列表进行排序的代码片段，它用一个匿名类创建了排序的比较函数（加强排列顺序）：

  ```java
  // Anonymous class instance as a function object - obsolete!
  Collection.sort(words, new Comparator<String>() {
    public int compare(String s1, String s2) {
      return Integer.compare(s1.length(), s2.length());
    }
  })
  ```

  ​	*匿名类满足了传统的面向对象的设计模式对函数对象的需求，最著名的有策略（Strategy）模式。 Comparator接口代表一种排序的抽象策略（abstract strategy）；上述的匿名类则是为字符串排序的一种具体策略(concrete strategy)。但是，匿名类的烦琐使得在Java中进行函数编程的前景变得十分黯淡。*

  ​	在Java8中，形成了"带有单个抽象方法的接口是特殊的，值得特殊对待"的观念。这些接口现在被称作**函数接口(functional interface)**，Java允许利用Lambda表达式(Lambda expression，简称 Lambda)创建这些接口的实例。 Lambda类似于匿名类的函数，但是比它简洁得多。以下是上述代码用 Lambda代替匿名类之后的样子。样板代码没有了，其行为也十分明确：

  ```java
  // Lambda expression as function object (replaces anonymous class)
  Collections.sort(words, (s1, s2) -> Integer.compare(s1.length(), s2.length()));
  ```

  ​	注意， Lambda的类型(`Comparator<String>`)、其参数的类型（s1和s2，两个都是String）及其返回值的类型(int)，都没有出现在代码中。<u>编译器利用一个称作类型推导(type inference)的过程，根据上下文推断岀这些类型。在某些情况下，编译器无法确定类型，你就必须指定</u>。类型推导的规则很复杂：在JLS[JLS，18]中占了整章的篇幅。几乎没有程序员能够详细了解这些规则，但是没关系。**删除所有 Lambda参数的类型吧，除非它们的存在能够使程序变得更加清晰**。如果编译器产生一条错误消息，告诉你无法推导出Lambda参数的类型，那么你就指定类型。<u>有时候还需要转换返回值或者整个 Lambda表达式，但是这种情况很少见</u>。

  ​	**关于类型推导应该增加一条警告。（E26）告诉你不要使用原生态类型，（E29）说过要支持泛型类型，（E30）说过要支持泛型方法**。<u>在使用 Lambda时，这条建议确实非常重要，因为编译器是从泛型获取到得以执行类型推导的大部分类型信息的。如果你没有提供这些信息，编译器就无法进行类型推导，你就必须在 Lambda中手工指定类型，这样极大地增加了它们的烦琐程度</u>。如果上述代码片段中的变量 words声明为原生态类型List，而不是参数化的类型`List<String>`，它就不会进行编译。

  ​	当然，如果用 Lambda表达式(详见E14和E43)代替比较器构造方法(comparator construction method)，有时这个代码片段中的比较器还会更加简练：

  ```java
  Collections.sort(words, comparingInt(String::length));
  ```

  ​	事实上，如果利用Java8在List接口中添加的sort方法，这个代码片段还可以更加简短一些：

  ```java
  words.sort(comparingInt(String::length));
  ```

  ​	Java中增加了 Lambda之后，使得之前不能使用函数对象的地方现在也能使用了。例如，以（E34）中的 Operation枚举类型为例。由于每个枚举的apply方法都需要不同的行为，我们用了特定于常量的类主体，并覆盖了每个枚举常量中的apply方法。通过以下代码回顾一下：

  ```java
  // Enum type with constant-specific class bodies & data (E34)
  public enum Operation {
    PLUS("+") {
      public double apply(double x, double y) { return x + y; }
    },
    MINUS("-") {
      public double apply(double x, double y) { return x - y; }
    },
    TIMES("*") {
      public double apply(double x, double y) { return x * y; }
    },
    DIVIDE("/") {
      public double apply(double x, double y) { return x / y; }
    };
    private final String symbol;
    Operation(String symbol) { this.symbol = symbol; }
    @Override
    public abstract double apply(double x, double y);
  }
  ```

  ​	由（E34）可知，枚举实例域优先于特定于常量的类主体。 Lambda使得利用前者实现特定于常量的行为变得比用后者来得更加容易了。只要给每个枚举常量的构造器传递一个实现其行为的 Lambda即可。构造器将 Lambda保存在一个实例域中，apply方法再将调用转给 Lambda。由此得到的代码比原来的版本更简单，也更加清晰：

  ```java
  // Enum with function object fields & constant-specific behavior
  public enum Operation {
    PLUS("+", (x, y) -> x + y),
    MINUS("-", (x, y) -> x - y),
    TIMES("*", (x, y) -> x * y),
    DIVIDE("/", (x, y) -> x /y);
    
    private final String symbol;
    private final DoubleBinaryOperator op;
    
    Operation(String symbol, DoubleBinaryOperator op) {
      this.symbol = symbol;
      this.op = op;
    }
    
    @Override
    public String toString() { return symbol; }
    
    public double apply(double x, double y) {
      return op.applyAsDouble(x, y);
    }
  }
  ```

  ​	*注意，这里给 Lambda使用了DoubleBinaryOperator接口，代表枚举常量的行为。这是在java.util.function（E44）中预定义的众多函数接口之一。它表示一个带有两个double参数的函数，并返回一个double结果。*

  ​	看看基于 Lambda的 Operation枚举，你可能会想，特定于常量的方法主体已经形同虚设了，但是实际并非如此。与方法和类不同的是， **Lambda没有名称和文档；如果一个计算本身不是自描述的，或者超出了几行，那就不要把它放在一个 Lambda中**。

  + <u>对于 Lambda而言，一行是最理想的，三行是合理的最大极限</u>。如果违背了这个规则，可能对程序的可读性造成严重的危害。
  + 如果 Lambda很长或者难以阅读，要么找一种方法将它简化，要么重构程序来消除它。
  + 而且，传入枚举构造器的参数是在静态的环境中计算的。因而，枚举构造器中的 Lambda无法访间枚举的实例成员。
  + **如果枚举类型带有难以理解的特定于常量的行为，或者无法在几行之内实现，又或者需要访问实例域或方法，那么特定于常量的类主体仍然是首选**。

  ​	同样地，你可能会认为，在 Lambda时代，匿名类已经过时了。这种想法比较接近事实，但是仍有一些工作用 Lambda无法完成，只能用匿名类才能完成。 

  + **Lambda限于函数接口。如果想创建抽象类的实例，可以用匿名类来完成，而不是用 Lambda**。

  + 同样地，可以用匿名类为带有多个抽象方法的接口创建实例。

  + **最后一点， Lambda无法获得对自身的引用**。

    + **<u>在 Lambda中，关键字this是指外围实例，这个通常正是你想要的</u>。**
    + **在匿名类中，关键字this是指匿名类实例。**

    **如果需要从函数对象的主体内部访问它，就必须使用匿名类**。

  ​	<u>Lambda与匿名类共享你无法可靠地通过实现来序列化和反序列化的属性</u>。因此，**尽可能不要(除非迫不得已)序列化一个 Lambda(或者匿名类实例)**。如果想要可序列化的函数对象，如Comparator，就使用私有静态嵌套类（E24）的实例。

---

+ 小结

  ​	总而言之，从Java8开始， Lambda就成了表示小函数对象的最佳方式。**千万不要给函数对象使用匿名类，除非必须创建非函数接口的类型的实例**。同时，还要记住， Lambda使得表示小函数对象变得如此轻松，因此打开了之前从未实践过的在Java中进行函数编程的大门。

## * E43 方法引用优先于Lambda

- 概述

  ​	与匿名类相比， Lambda的主要优势在于更加简洁。Java提供了生成比 Lambda更简洁函数对象的方法：方法引用(method reference)。

  ​	以下代码片段的源程序是用来保持从任意键到Integer值的一个映射。如果这个值为该键的实例数目，那么这段程序就是一个多集合的实现。这个代码片段的作用是，当这个键不在映射中时，将数字1和键关联起来；或者当这个键已经存在，就负责递增该关联值：

  ```java
  map.merge(key, 1, (count, incr) -> count + incr);
  ```

  ​	注意，这行代码中使用了merge方法，这是Java8版本在Map接口中添加的。如果指定的键没有映射，该方法就会插入指定值；如果有映射存在， merge方法就会将指定的函数应用到当前值和指定值上，并用结果覆盖当前值。这行代码代表了merge方法的典型用例。

  ​	这样的代码读起来清晰明了，但仍有些样板代码。参数 count和incr没有添加太多价值，却占用了不少空间。实际上， Lambda要告诉你的就是，该函数返回的是它两个参数的和。<u>从Java8开始， Integer(以及所有其他的数字化基本包装类型都)提供了一个名为sum的静态方法，它的作用也同样是求和</u>。我们只要传入一个对该方法的引用，就可以更轻松地得到相同的结果

  ```java
  map.merge(key, 1, Integer::sum);
  ```

  ​	方法带的参数越多能用方法引用消除的样板代码就越多。但在有些 Lambda中，即便它更长，但你所选择的参数名称提供了非常有用的文档信息，也会使得 Lambda的可读性更强，并且比方法引用更易于维护。

  ​	**只要方法引用能做的事，就没有 Lambda不能完成的(只有一种情况例外，有兴趣的读者请参见JLS，9.9-2)**。也就是说，使用方法引用通常能够得到更加简短、清晰的代码。

  ​	<u>如果Lambda太长，或者过于复杂，还有另一种选择：从 Lambda中提取代码，放到一个新的方法中，并用该方法的一个引用代替Lambda</u>。你可以给这个方法起一个有意义的名字，并用自己满意的方式编写进入文档。

  ​	如果是用IDE编程，则可以在任何可能的地方都用方法引用代替 Lambda。通常(但并非总是)应该让IDE把握机会好好表现一下。有时候， Lambda也会比方法引用更加简洁明了。这种情况大多是当方法与 Lambda处在同一个类中的时候。比如下面的代码片段，假定发生在一个名为CoshThisClassNameIsHumongous的类中：

  ```java
  service.execute(CoshThisClassNameIsHumongous::action);
  ```

  ​	Lambda版本的代码如下：

  ```java
  service.execute(() -> action());
  ```

  ​	这个代码片段使用了方法引用，但是它既不比 Lambda更简短，也不比它更清晰，因此应该优先考虑 Lambda。类似的还有 Function接口，它用一个静态工厂方法返回id函数`Function.identity()`如果它不用这个方法，而是在行内编写同等的 Lambda表达式`x->x`，一般会比较简洁明了。

  ​	许多方法引用都指向静态方法，但其中有4种没有这么做。其中两个是有限制(bound)和无限制(unbound)的实例方法引用。

  + 在有限制的引用中，接收对象是在方法引用中指定的。有限制的引用本质上类似于静态引用：函数对象与被引用方法带有相同的参数。
  + 在无限制的引用中，接收对象是在运用函数对象时，通过在该方法的声明函数前面额外添加一个参数来指定的。无限制的引用经常用在流管道(Stream pipeline)（E45）中作为映射和过滤函数。
  + 最后，还有两种构造器(constructor)引用，分别针对类和数组。构造器引用是充当工厂对象。

  ​	这五种方法引用概括如下:

  | 方法引用类型 | 范例                   | Lambda等式                                              |
  | ------------ | ---------------------- | ------------------------------------------------------- |
  | 静态         | Integer::parseInt      | str -> Integer.parseInt(str);                           |
  | 有限制       | Instant.now()::isAfter | Instant then = Instant.now();<br />t -> then.isAfter(t) |
  | 无限制       | String::toLowerCase    | str -> str.toLowerCase()                                |
  | 类构造器     | TreeMap<K,V>::new      | () -> new TreeMap<K , V>                                |
  | 数组构造器   | int[]::new             | len -> new int[len]                                     |

---

- 小结

  ​	总而言之，方法引用常常比Lambda表达式更加简洁明了。

  ​	**只要方法引用更加简洁、清晰，就用方法引用；如果方法引用并不简洁，就坚持使用 Lambda**。

## * E44 坚持使用标准的函数接口

+ 概述

  ​	在Java具有 Lambda表达式之后，编写API的最佳实践也做了相应的改变。例如在模板方法(Template Method)模式中，用一个子类覆盖基本类型方法(primitive method)，来限定其超类的行为，这是最不讨人喜欢的。现在的替代方法是提供一个接受函数对象的静态工厂或者构造器，便可达到同样的效果。在大多数情况下，需要编写更多的构造器和方法，以函数对象作为参数。需要非常谨慎地选择正确的函数参数类型。

  ​	以LinkedHashMap为例。每当有新的键添加到映射中时，put就会调用其受保护的removeEldestEntry方法。如果覆盖该方法，便可以用这个类作为缓存。当该方法返回true，映射就会删除最早传入该方法的条目。下列覆盖代码允许映射增长到100个条目，然后每添加一个新的键，就会删除最早的那个条目，始终保持最新的100个条目：

  ```java
  protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return size() > 100;
  }
  ```

  ​	这个方法很好用，但是用 Lambda可以完成得更漂亮。假如现在编写 LinkedhashMap，它会有一个带函数对象的静态工厂或者构造器。看一下 removeEldestEntry的声明，你可能会以为该函数对象应该带一个Map.Entry\<K，V\>，并且返回一个 boolean，但实际并非如此：removeEldestEntry方法会调用size()，获取映射中的条目数量，这是因为 removeEldestEntry是映射中的一个实例方法。传到构造器的函数对象则不是映射中的实例方法，无法捕捉到，因为调用其工厂或者构造器时，这个映射还不存在。所以，映射必须将它自身传给函数对象，因此必须传入映射及其最早的条目作为 remove方法的参数。声明一个这样的函数接口的代码如下：

  ```java
  // Unneccessary functional interface; use a standard one instead.
  @FunctionalInterface interface EledestEntryRomovalFunction<K,V> {
    boolean remove(Map<K,V> map, Map.Entry<K,V> eldest);
  }
  ```

  ​	这个接口可以正常工作，但是不应该使用，因为没必要为此声明一个新的接口。`java.util.function`包已经为此提供了大量标准的函数接口。**只要标准的函数接口能够满足需求，通常应该优先考虑，而不是专门再构建一个新的函数接口**。这样会使API更加容易学习，通过减少它的概念内容，显著提升互操作性优势，因为许多标准的函数接口都提供了有用的默认方法。如 Predicate接口提供了合并断言的方法。对于上述 LinkedHashMap范例，应该优先使用标准的`BiPredicate<Map<K,V>，Map.Entry<K,V>>`接口，而不是定制EldestEntryRemovalFunction接口。

  ​	<u>java.util.Function中共有43个接口。别指望能够全部记住它们，但是如果能记住其中6个基础接口，必要时就可以推断出其余接口了</u>。

  + 基础接口作用于对象引用类型。

  + Operator接口代表其结果与参数类型一致的函数。 
  + Predicate接口代表带有一个参数并返回一个 boolean的函数。
  + Function接口代表其参数与返回的类型不一致的函数。
  + Supplier接口代表没有参数并且返回(或"提供")一个值的函数。
  + 最后， Consumer代表的是带有一个参数但不返回任何值的函数，相当于消费掉了其参数。

  这6个基础函数接口概述如下：

  | 接口                | 函数签名            | 范例                |
  | ------------------- | ------------------- | ------------------- |
  | UnaryOperator\<T\>  | T apply(T t)        | String::toLowerCase |
  | BinaryOperator\<T\> | T apply(T t1, T t2) | BigInteger::add     |
  | Predicate\<T\>      | boolean test(T t)   | Collection::isEmpty |
  | Function\<T,R\>     | R apply(T t)        | Arrays::asList      |
  | Supplier\<T\>       | T get()             | Instant::now        |
  | Comsumer\<T\>       | void accept(T t)    | System.out::println |

  ​	这6个基础接口各自还有3种变体，分别可以作用于基本类型int、long和double。它们的命名方式是在其基础接口名称前面加上基本类型而得。因此，以带有int的 predicate接口为例，其变体名称应该是 IntPredicate，LongBinaryOperator是一个二进制运算符带有两个long值参数并返回一个long值。这些变体接口的类型都不是参数化的，除 Function变体外，后者是以返回类型作为参数。例如，`LongFunction<int[]>`表示带有一个long参数，并返回一个int[]数组。

  ​	Function接口还有9种变体，用于结果类型为基本类型的情况。源类型和结果类型始终不一样，因为从类型到自身的函数就是UnaryOperator。如果源类型和结果类型均为基本类型，就是在 Function前面添加格式如 ScrToResult，如 LongToIntFunction(有6种变体)。如果源类型为基本类型，结果类型是一个对象参数，则要在 Function前添加`<Src> ToObj`，如`DoubleToObjFunction`(有3种变体)。

  ​	这三种基础函数接口还有带两个参数的版本，如BiPredicate<T，U>、 BiFunction<T，U，R>和 BiConsumer<T，U>。还有BiFunction变体用于返回三个相关的基本类型：ToIntBiFunction<T， U>，ToLongBiFunction<T， U> 和ToDoubleBiFunction<T，U>。Consumer接口也有带两个参数的变体版本，它们带一个对象引用和一个基本类型：ObjDoubleConsumer\<T\>、 ObjIntConsumer\<T\>和ObjLongConsumer\<T\>。总之，这些基础接口有9种带两个参数的版本。

  ​	最后，还有 BooleanSupplier接口，它是 Supplier接口的一种变体，返回 boolean值。这是在所有的标准函数接口名称中唯一显式提到 boolean类型的，但 boolean返回值是通过 Predicate及其4种变体来支持的。 BooleanSupplier接口和上述段落中提及的42个接口，总计43个标准函数接口。显然，这是个大数目，但是它们之间并非纵横交错。另一方面，你需要的函数接口都替你写好了，它们的名称都是循规蹈矩的，需要的时候并不难找到。

  ​	现有的大多数标准函数接口都只支持基本类型。**千万不要用带包装类型的基础函数接口来代替基本函数接口**。虽然可行，但它破坏了第61条的规则"基本类型优于裝箱基本类型"。使用装箱基本类型进行批量操作处理，最终会导致致命的性能问题。

  ​	现在知道了，通常应该优先使用标准的函数接口，而不是用自己编写的接口。但什么时候应该自己编写接口呢？当然，是在如果没有任何标准的函数接口能够满足你的需求之时，如需要一个带有三个参数的 Predicate接口，或者需要一个抛出受检异常的接口时，当然就需要自己编写啦。但是也有这样的情况：有结构相同的标准函数接口可用，却还是应该自己编写函数接口。

  ​	还是以咱们的老朋友 Comparator\<T>为例吧。它与 ToIntBiFunction<T，T>接口在结构上一致，虽然前者被添加到类库中时，后一个接口已经存在，但如果用后者就错了。 Comparator之所以需要有自己的接口，有三个原因。首先，每当在API中使用时，其名称提供了良好的文档信息，并且被大量使用。其次， Comparator接口对于如何构成个有效的实例，有着严格的条件限制，这构成了它的总则(general contract)。实现该接口相当于承诺遵守其契约。第三，这个接口配置了大量很好用的缺省方法，可以对比较器进行转换和合并。

  ​	**如果你所需要的函数接口与 Comparator一样具有一项或者多项以下特征，则必须认真考虑自己编写专用的函数接口，而不是使用标准的函数接口**：

  + 通用，并且受益于描述性的名称。
  + 具有与其关联的严格的契约。
  + 将受益于定制的缺省方法。

  ​	如果决定自己编写函数接口，一定要记住，它是一个接口，因而设计时应当万分谨慎（E21）。

  ​	注意，EldestEntryRemovalFunction接口(详见第199页)是用`@FunctionalInterface`注解进行标注的。这个注解类型本质上与@Override类似。这是一个标注了程序员设计意图的语句，它有三个目的：

  + 告诉这个类及其文档的读者，这个接口是针对Lambda设计的；
  + 这个接口不会进行编译，除非它只有一个抽象方法；
  + 避免后续维护人员不小心给该接口添加抽象方法。

  ​	**必须始终用@Functionallnterface注解对自己编写的函数接口进行标注**。

  ​	**最后一点是关于函数接口在API中的使用。不要在相同的参数位置，提供不同的函数接口来进行多次重载的方法，否则可能在客户端导致歧义**。这不仅仅是理论上的问题。比如ExecutorService的submit方法就可能带有Callable\<T\>或者 Runnable，并且还可以编写一个客户端程序，要求进行一次转换，以显示正确的重载（E52）。避免这个问题的最简单方式是，不要编写在同一个参数位置使用不同函数接口的重载。这是该建议的一个特例，详情请见（E52）。

---

+ 小结

  ​	总而言之，既然Java有了 Lambda，就必须时刻谨记用 Lambda来设计API。输入时接受函数接口类型，并在输出时返回之。一般来说，最好使用java.util.function.Function中提供的标准接口，但是必须警惕在相对罕见的几种情况下，最好还是自己编写专用的函数接口。

## * E45 谨慎使用Stream

+ 概述

  ​	在Java8中增加了 Stream APl，简化了串行或并行的大批量操作。这个API提供了两个关键抽象：Stream（流）代表数据元素有限或无限的顺序， Stream pipeline（流管道）则代表这些元素的一个多级计算。 Stream中的元素可能来自任何位置。常见的来源包括集合数组、文件、正则表达式模式匹配器、伪随机数生成器，以及其他 Stream。 Stream中的数据元素可以是对象引用，或者基本类型值。它支持三种基本类型：int、long和 double。

  ​	一个 Stream pipeline中包含一个源 Stream，接着是0个或者多个中间操作(intermediate operation)和一个终止操作(terminal operation)。每个中间操作都会通过某种方式对Stream进行转换，例如将每个元素映射到该元素的函数，或者过滤掉不满足某些条件的所有元素。所有的中间操作都是将一个 Stream转换成另一个 Stream，其元素类型可能与输入的 Stream一样，也可能不同。终止操作会在最后一个中间操作产生的 Strean上执行一个最终的计算，例如将其元素保存到一个集合中，并返回某一个元素，或者打印出所有元素等。

  ​	**Stream pipeline通常是lazy的：直到调用终止操作时才会开始计算，对于完成终止操作不需要的数据元素，将永远都不会被计算**。正是这种lazy计算，使无限 Stream成为可能。注意，没有终止操作的Stream pipeline将是一个静默的无操作指令，因此千万不能忘记终止操作。

  ​	Stream APl是流式(fluent)的：所有包含 pipeline的调用可以链接成一个表达式。事实上，多个 pipeline也可以链接在一起，成为一个表达式。

  ​	**在默认情况下， Stream pipeline是按顺序运行的。要使 pipeline并发执行，只需在该pipeline的任何 Stream上调用parallel方法即可，但是通常不建议这么做（E48）**。

  ​	Stream API包罗万象，足以用Stream执行任何计算，但是"可以"并不意味着"应该"。如果使用得当， Stream可以使程序变得更加简洁、清晰；如果使用不当，会使程序变得混乱且难以维护。对于什么时候应该使用Stream，并没有硬性的规定，但是可以有所启发。

  ​	以下面的程序为例，它的作用是从词典文件中读取单词，并打印出单词长度符合用户指定的最低值的所有换位词。记住，包含相同的字母，但是字母顺序不同的两个词，称作换位词(anagram)。该程序会从用户指定的词典文件中读取每一个词，并将符合条件的单词放入一个映射中。这个映射键是按字母顺序排列的单词，因此"staple"的键是"aelpst"，"petals"的键也是"aelpst"：这两个词就是换位词，所有换位词的字母排列形式是一样的(有时候也叫 alphagram)。映射值是包含了字母排列形式一致的所有单词。词典读取完成之后，每一个列表就是一个完整的换位词组。随后，程序会遍历映射的values()，预览并打印出单词长度符合极限值的所有列表。

  ```java
  // Prints all large anagram groups in a dictionary iteratively
  public class Anagrams {
    public static void main(String[] args) throws IOException {
      File dictionary = new File(args[0]);
      int minGroupSize = Integer.parseInt(args[1]);
      
      Map<String, Set<String>> groups = new HashMap<>();
      try(Scanner s = new Scanner(dictionary)) {
        while (s.hasNext()) {
          String word = s.next();
          groups.computeIfAbsent(alphabetize(word),
             (unused) -> new TreeSet<>()).add(word);
        }
      }
      for(Set<String> group : groups.values())
        if(group.size() >= minGroupSize)
          System.out.println(group.size() + ": " + group);
    }
    
    private static String alphabetize(String s) {
      char[] a = s.toCharArray();
      Arrays.sort(a);
      return new String(a);
    }
  }
  ```

  ​	这个程序中有一个步骤值得注意。被插入到映射中的每一个单词都以粗体显示，这是使用了Java8中新增的 computeIfAbsent方法。<u>这个方法会在映射中查找一个键：如果这个键存在，该方法只会返回与之关联的值。如果键不存在，该方法就会对该键运用指定的函数对象算出一个值，将这个值与键关联起来，并返回计算得到的值。 computeIfAbsent方法简化了将多个值与每个键关联起来的映射实现</u>。

  ​	下面举个例子，它也能解决上述问题，只不过大量使用了 Stream。注意，它的所有程序都是包含在一个表达式中，除了打开词典文件的那部分代码之外。<u>之所以要在另一个表达式中打开词典文件，只是为了使用try-with-resources语句，它可以确保关闭词典文件</u>：

  ```java
  // Overuse of streams - don't do this!
  public class Anagrams {
    public static void main(String[] args) throws IOException {
      Path dictionary = Paths.get(args[0]);
      int minGroupSize = Integer.parseInt(args[1]);
  
      try(Stream<String> words = Files.lines(dictionary)) {
        words.collect(
          groupingBy(word -> word.char().sorted()
                     .collect(StringBuilder::new,
                              (sb, c) -> sb.append((char) c),
                              StringBuilder::apend).toString()))
          .values().stream()
          .filter(group -> group.size() >= minGroupSize)
          .map(group -> group.size() + ": " + group)
          .forEach(System.out::println);
      }
    }
  }
  ```

  ​	如果你发现这段代码好难懂，别担心，你并不是唯一有此想法的人。它虽然简短，但是难以读懂，对于那些使用 Stream还不熟练的程序员而言更是如此。滥用 Stream会使程序代码更难以读懂和维护。

  ​	好在还有一种舒适的中间方案。下面的程序解决了同样的问题，它使用了 Stream，但是没有过度使用。结果，与原来的程序相比，这个版本变得既简短又清晰：

  ```java
  // Tasteful ues of streams enhances clarity and conciseness
  public class Anagrams {
    public static void main(String[] args) throws IOException {
      Path dictionary = Paths.get(args[0]);
      int minGroupSize = Integer.parseInt(agrs[1]);
      
      try(String<String> words = Files.lines(dictionary)) {
        words.collect(groupingBy(word -> alphabetize(word)))
          .values().stream()
          .filter(group -> group.size() >= minGroupSize)
          .forEach(g -> System.out.println(g.size() + ": " + g));
      }
    }
    // alphabetize method is the same as in original version
  }
  ```

  ​	即使你之前没怎么接触过 Stream，这段程序也不难理解。它在try-with-resources块中打开词典文件，获得一个包含了文件中所有代码的 Stream。Stream变量命名为 words，是建议Stream中的每个元素均为单词。这个 Stream中的 pipeline没有中间操作；它的终止操作将所有的单词集合到一个映射中，按照它们的字母排序形式对单词进行分组（E46）。这个映射与前面两个版本中的是完全相同的。随后，在映射的values()视图中打开了一个新的`Stream<List<String>`。当然，这个 Stream中的元素都是换位词分组。 Strean进行了过滤，把所有单词长度小于 minGroupSize的单词都去掉了，最后，通过终止操作的 foreach打印出剩下的分组。

  ​	注意， Lambda参数的名称都是经过精心挑选的。实际上参数应当以group命名，只是这样得到的代码行对于书本而言太宽了。**在没有显式类型的情况下，仔细命名 Lambda参数，这对于 Stream pipeline的可读性至关重要**。

  ​	还要注意单词的字母排序是在一个单独的alphabetize方法中完成的。给操作命名，并且不要在主程序中保留实现细节，这些都增强了程序的可读性。**在 Stream pipeline中使用 helper方法，对于可读性而言，比在迭代化代码中使用更为重要**，因为 pipeline缺乏显式的类型信息和具名临时变量。

  ​	可以重新实现alphabetize方法来使用 Stream，只是基于 Stream的 alphabetize方法没那么清晰，难以正确编写，速度也可能变慢。这些不足是因为<u>Java不支持基本类型的 char Stream(这并不意味着Java应该支持 char stream；也不可能支持)。为了证明用Stream处理char值的各种危险，请看以下代码</u>：

  ```java
  "Hello world!".chars().forEach(System.out::print);
  ```

  ​	或许你以为它会输出`Hello world!`，但是运行之后发现，它输出的是721011081081113211911111410810033。<u>这是因为`"Hello world!".chars()`返回的 Stream中的元素，并不是char值，而是int值，因此调用了print的int覆盖</u>。名为 chars的方法，却返回int值的 Stream，这固然会造成困扰。修正方法是利用转换强制调用正确的覆盖：

  ```java
  Hello world!".chars(， foreach(x-> System. out. print((char) x));
  ```

  ​	<u>但是，**最好避免利用 Stream来处理char值**。刚开始使用 Stream时，可能会冲动到恨不得将所有的循环都转换成 Stream，但是切记，千万别冲动。这可能会破坏代码的可读性和易维护性。一般来说，即使是相当复杂的任务，最好也结合 Stream和迭代来一起完成，如上面的 Anagrams程序范例所示。因此，**重构现有代码来使用 Stream，并且只在必要的时候才在新代码中使用**</u>。

  ​	如本条目中的范例程序所示， Stream pipeline利用函数对象(一般是 Lambda或者方法引用)来描述重复的计算，而迭代版代码则利用代码块来描述重复的计算。下列工作只能通过代码块，而不能通过函数对象来完成：

  + 从代码块中，可以读取或者修改范围内的任意局部变量；从 Lambda则只能读取final或者有效的final变量[JLS 4.12.4]，并且不能修改任何local变量。
  + 从代码块中，可以从外围方法中 return、 break或 continue外围循环，或者抛出该方法声明要抛出的任何受检异常；从 Lambda中则完全无法完成这些事情。

  ​	如果某个计算最好要利用上述这些方法来描述，它可能并不太适合 Stream。反之， Stream可以使得完成这些工作变得易如反掌：

  + 统一转换元素的序列
  + 过滤元素的序列
  + 利用单个操作(如添加、连接或者计算其最小值)合并元素的顺序
  + 将元素的序列存放到一个集合中，比如根据某些公共属性进行分组
  + 搜索满足某些条件的元素的序列

  ​	如果某个计算最好是利用这些方法来完成，它就非常适合使用 Stream。

  ​	<u>**利用 Stream很难完成的一件事情就是，同时从一个 pipeline的多个阶段去访问相应的元素：一旦将一个值映射到某个其他值，原来的值就丢失了**。一种解决办法是将每个值都映射到包含原始值和新值的一个对象对( pair object)，不过这并非万全之策，当 pipeline的多个阶段都需要这些对象对时尤其如此。这样得到的代码将是混乱、繁杂的，违背了 Strean的初衷。最好的解决办法是，当需要访问较早阶段的值时，将映射颠倒过来</u>。

  ​	例如，编写一个打印出前20个梅森素数(Mersenne primes)的程序。解释一下，梅森素数是一个形式为2<sup>p</sup>-1的数字。如果p是一个素数，相应的梅森数字也是素数；那么它就是一个梅森素数。作为 pipeline的第一个 Stream，我们想要的是所有素数。下面的方法将返回(无限) Stream。假设使用的是静态导入，便于访问 BigInteger的静态成员：

  ```java
  static Stream<BigInteger> primes() {
    return Stream.interate(TWO, BigInteger::nextProbablePrime);
  }
  ```

  ​	方法的名称(primes)是一个复数名词，它描述了 Stream的元素。强烈建议返回Stream的所有方法都采用这种命名惯例，因为可以增强 Stream pipeline的可读性。该方法使用静态工厂Stream.iterate，它有两个参数：Stream中的第一个元素，以及从前一个元素中生成下一个元素的一个函数。下面的程序用于打印出前20个梅森素数。

  ```java
  public static void main(String[] args) {
    primes().map(p -> TWO.pow(p.intValueExact()).subtract(ONE))
      .filter(mersenne -> mersenne.isProbablePrime(50))
      .limit(20)
      .forEach(System.out.println);
  }
  ```

  ​	这段程序是对上述内容的简单编码示范：它从素数开始，计算岀相应的梅森素数，过滤掉所有不是素数的数字(其中50是个神奇的数字，它控制着这个概率素性测试)，限制最终得到的 Stream为20个元素，并打印出来。

  ​	现在假设想要在每个梅森素数之前加上其指数(p)。这个值只出现在第一个 Stream中，因此在负责输出结果的终止操作中是访问不到的。所幸将发生在第一个中间操作中的映射颠倒过来，便可以很容易地计算出梅森数字的指数。该指数只不过是一个以二进制表示的位数，因此终止操作可以产生所要的结果

  ```java
  .foreach(mp -> System.out.println(mp.bitLength()+ ": " + mp));
  ```

  ​	现实中有许多任务并不明确要使用 Stream，还是用迭代。例如有个任务是要将一副新纸牌初始化。假设Card是一个不变值类，用于封装Rank和Suit，这两者都是枚举类型。这项任务代表了所有需要计算从两个集合中选择所有元素对的任务。数学上称之为两个集合的笛卡尔积。这是一个迭代化实现，嵌入了一个for-each循环，大家对此应当都非常熟悉了：

  ```java
  // Iterative Cartesian product computation
  private static List<Card> newDeck() {
    List<Card> result = new ArrayList<>();
    for(Suit suit : Suit.values())
      for(Rank rank : Rank.values())
        result.add(new Card(suit, rank));
    return result;
  }
  ```

  ​	这是一个基于 Stream的实现，利用了中间操作 flatMap。这个操作是将 Stream中的每个元素都映射到一个 Stream中，然后将这些新的 Stream全部合并到一个 Stream(或者将它们扁平化)。注意，这个实现中包含了一个嵌入式的 Lambda，如以下粗体部分所示：

  ```java
  // Stream-based Cartesian product computation
  private static List<Card> newDeck() {
    return Stream.of(Suit.values())
      .flatMap(suit -> 
               Stream.of(Rank.values())
               .map(rank -> new Card(suit, rank)))
      .collect(toList());
  }
  ```

  ​	这两种 new Deck版本哪一种更好？这取决于个人偏好，以及编程环境。第一种版本比较简单，可能感觉比较自然，大部分Java程序员都能够理解和维护，但是有些程序员可能会觉得第二种版本(基于 Stream的)更舒服。这个版本可能更简洁一点，如果已经熟练掌握 Stream和函数编程，理解起来也不难。如果不确定要用哪个版本，或许选择迭代化版本会更加安全一些。如果更喜欢 Stream版本，并相信后续使用这些代码的其他程序员也会喜欢，就应该使用 Stream版本。

---

+ 小结

  ​	总之，有些任务最好用 Stream完成，有些则要用迭代。而有许多任务则最好是结合使用这两种方法来一起完成。具体选择用哪一种方法，并没有硬性、速成的规则，但是可以参考一些有意义的启发。在很多时候，会很清楚应该使用哪一种方法;有些时候，则不太明显。**如果实在不确定用 Stream还是用迭代比较好，那么就两种都试试，看看哪一种更好用吧**

## E46 优先选择Stream中无副作用的函数

+ 概述

  ​	如果刚接触 Stream，可能比较难以掌握其中的窍门。就算只是用 Stream pipeline来表达计算就困难重重。当你好不容易成功了，运行程序之后，却可能感到这么做并没有享受到多大益处。 Strean并不只是一个API，它是一种基于函数编程的模型。为了获得 Strean带来的描述性和速度，有时还有并行性，必须采用范型以及API。

  ​	Stream范型最重要的部分是把计算构造成一系列变型，每一级结果都尽可能靠近上级结果的纯函数(pure function)。纯函数是指其结果只取决于输入的函数：它不依赖任何可变的状态，也不更新任何状态。为了做到这一点，传入Stream操作的任何函数对象，无论是中间操作还是终止操作，都应该是无副作用的。

  ​	有时会看到如下代码片段，它构建了一张表格，显示这些单词在一个文本文件中出现的频率：

  ```java
  // Uses the streams API but not the paradigm--Don't do this!
  Map<String, Long> freq = new HashMap<>();
  try (Stream<String> word = new Scanner(file).tokens()) {
    words.forEach(word -> {
      freq.merge(word.toLowerCase(), 1L, Long::sum);
    });
  }
  ```

  ​	以上代码有什么问题吗？它毕竟使用了 Strean、 Lambda和方法引用，并且得出了正确的答案。简而言之，这根本不是 Stream代码；只不过是伪装成 Stream代码的迭代式代码。它并没有享受到 Stream API带来的优势，代码反而更长了点，可读性也差了点，并且比相应的迭代化代码更难维护。因为这段代码利用一个改变外部状态(频率表)的 Lambda，完成了在终止操作的 foreach中的所有工作。 foreach操作的任务不只展示由 Stream执行的计算结果，这在代码中并非好事，改变状态的 Lambda也是如此。那么这段代码应该是什么样的呢？

  ```java
  // Proper ues of streams to initialize a frequency table
  Map<String, Long> freq;
  try(Stream<String> words = new Scanner(file).tokens()) {
    freq = words.collect(groupingBy(String::toLowerCase, counting()));
  }
  ```

  ​	这个代码片段的作用与前一个例了一样，只是正确使用了 Stream API，变得更加简洁、清晰。那么为什么有人会以其他的方式编写呢？这是为了使用他们已经熟悉的工具。Java程序员都知道如何使用 for-each循环，终止操作的forEach也与之类似。但 forEach操作是终止操作中最low的，也是对 Stream最不友好的。它是显式迭代，因而不适合并行。 **forEach操作应该只用于报告 Stream计算的结果，而不是执行计算**。有时候，也可以将 forEach用于其他目的，比如将 Stream计算的结果添加到之前已经存在的集合中去。

  ​	改进过的代码使用了一个收集器(collector)，为了使用 Stream，这是必须了解的一个新概念。Collectors API很吓人：它有39种方法，其中有些方法还带有5个类型参数！好消息是，你不必完全搞懂这个API就能享受它带来的好处。对于初学者，可以忽略Cllector接口，并把收集器当作封装缩减策略的一个黑盒子对象。在这里，缩减的意思是将Stream的元素合并到单个对象中去。收集器产生的对象一般是一个集合(即名称收集器)。

  ​	将Stream的元素集中到一个真正的Collection里去的收集器比较简单。有三个这样的收集器：toList()、 toSet()和toCollection(collectionFactory)。它们分别返回一个列表、一个集合和程序员指定的集合类型。了解了这些，就可以编写Stream pipeline，从频率表中提取排名前十的单词列表了：

  ```java
  // Pipeline to get a top-ten list of words from a frequency table
  List<String> topTen = freq.keySet().stream()
    .sorted(comparing(freq::get).reversed())
    .limit(10)
    .collect(toList())
  ```

  ​	注意，这里没有给 toList方法配上它的Collectors类。**静态导入Collectors的所有成员是惯例也是明智的，因为这样可以提升 Stream pipeline的可读性**。

  ​	这段代码中唯一有技巧的部分是传给 sorted的比较器 `comparing(freq::get).reversed()`。comparing方法是一个比较器构造方法（E14），它带有一个键提取函数。函数读取一个单词，"提取"实际上是一个表查找：有限制的方法引用freq:get在频率表中查找单词，并返回该单词在文件中出现的次数。最后，在比较器上调用 reversed，按频率高低对单词进行排序。后面的事情就简单了，只要限制 Stream为10个单词，并将它们集中到一个列表中即可。

  ​	上一段代码是利用Scanner的Stream方法来获得Stream。这个方法是在Java9中增加的。如果使用的是更早的版本，可以把实现 Iterator的扫描器，翻译成使用了类似于（E47）中适配器的 Stream(streamOf( Iterable\<E\>))。

  ​	Collectors中的另外36种方法又是什么样的呢？它们大多数是为了便于将 Stream集合到映射中，这远比集中到真实的集合中要复杂得多。每个 Stream元素都有一个关联的键和值，多个 Stream元素可以关联同一个键。

  ​	最简单的映射收集器是toMap(keyMapper，valueMapper)，它带有两个函数，其中一个是将 Stream元素映射到键，另一个是将它映射到值。我们采用（E34）fromString实现中的收集器，将枚举的字符串形式映射到枚举本身：

  ```java
  // Using a toMap collector to make a map from string to enum
  private static final Map<String, Operation> stringToEnum = 
    Stream.of(values()).collect(
  toMap(Object::toString, e -> e));
  ```

  ​	如果Stream中的每个元素都映射到一个唯一的键，那么这个形式简单的 toMap是很完美的。如果多个 Stream元素映射到同一个键， pipeline就会抛出一个IllegalStateException异常将它终止。

  ​	toMap更复杂的形式，以及groupingBy方法，提供了更多处理这类冲突的策略。其中一种方式是除了给 toMap方法提供了键和值映射器之外，还提供一个合并函数(merge function)。合并函数是一个 BinaryOperator\<V\>，这里的V是映射的值类型。合并函数将与键关联的任何其他值与现有值合并起来，因此，假如合并函数是乘法，得到的值就是与该值映射的键关联的所有值的积。

  ​	带有三个参数的toMap形式，对于完成从键到与键关联的被选元素的映射也是非常有用的。假设有一个 Stream，代表不同歌唱家的唱片，我们想得到一个从歌唱家到最畅销唱片之间的映射。下面这个收集器就可以完成这项任务。

  ```java
  // Collector to generate a map from key to chosen element for key
  Map<Artist, Album> toHits = albums.collect(
  toMap(Album::artist, a->a, maxBy(comparing(Album::sales))));
  ```

  ​	注意，这个比较器使用了静态工厂方法 maxBy，这是从 BinaryOperator静态导入的。该方法将 Comparator\<T\>转换成一个 BinaryOperator\<T\>，用于计算指定比较器产生的最大值。在这个例子中，比较器是由比较器构造器方法 comparing返回的，它有个键提取函数`Album::sales`。这看起来有点绕，但是代码的可读性良好。不严格地说，它的意思是"将唱片的 Stream转换成一个映射，将每个歌唱家映射到销量最佳的唱片"。这就非常接近问题陈述了。

  ​	带有三个参数的 tmap形式还有另一种用途,即生成一个收集器,当有冲突时强制保留最后更新”(last- write-wins)。对于许多 Stream而言,结果是不确定的,但如果与映射函数的键关联的所有值都相同,或者都是可接受的,那么下面这个收集器的行为就正是你所要的：

  ```java
  // Collector to impose last-write-wins policy
  toMap(keyMapper, valuesMapper, (oldVal, newVal) -> newVal)
  ```

  ​	toMap的第三个也是最后一种形式是，带有第四个参数，这是一个映射工厂，在使用时要指定特殊的映射实现，如 EnumMap或者 TreeMap。

  ​	tmap的前三种版本还有另外的变换形式，命名为 toconcurrentmap，能有效地并行运行，并生成 Concurrenthashmap实例。

  ​	除了toMap方法，Collectors API还提供了groupingBy方法，它返回收集器以生成映射，根据分类函数将元素分门别类。分类函数带有一个元素，并返回其所属的类别。这个类别就是元素的映射键。 groupingBy方法最简单的版本是只有一个分类器，并返回个映射，映射值为每个类别中所有元素的列表。下列代码就是在（E45）的 Anagram程序中用于生成映射(从按字母排序的单词，映射到字母排序相同的单词列表)的收集器：

  ```java
  words.collect(groupingBy(word -> alphabetize(word)))
  ```

  ​	如果要让groupingBy返回一个收集器，用它生成一个值而不是列表的映射，除了分类器之外，还可以指定一个下游收集器(downstream collector)。下游收集器从包含某个类别中所有元素的 Stream中生成一个值。这个参数最简单的用法是传入 toSet()，结果生成一个映射，这个映射值为元素集合而非列表。

  ​	另一种方法是传人toCollection(collectionFactory)，允许创建存放各元素类别的集合。这样就可以自由选择自己想要的任何集合类型了。带两个参数的groupingBy版本的另一种简单用法是，传入counting()作为下游收集器。这样会生成一个映射，它将每个类别与该类别中的元素数量关联起来，而不是包含元素的集合。这正是在本条目开头处频率表范例中见到的：

  ```java
  Map<String, Long> freq = words
    .collect(groupingBy(String::toLowerCase, couting()));
  ```

  ​	groupingBy的第三个版本，除下游收集器之外，还可以指定一个映射工厂。*注意，这个方法违背了标准的可伸缩参数列表模式：参数mapFactory要在downStream参数之前，而不是在它之后*。 groupingBy的这个版本可以控制所包围的映射，以及所包围的集合，因此，比如可以定义一个收集器，让它返回值为TreeSets的TreeMap。

  ​	groupingByConcurrent方法提供了 groupingBy所有三种重载的变体。这些变体可以有效地并发运行，生成 ConcurrentHashMap实例。还有一种比较少用到的 groupingBy变体叫作 partitioningBy。除了分类方法之外，它还带一个断言(predicate)，并返回一个键为Boolean的映射。这个方法有两个重载，其中一个除了带有断言之外，还带有下游收集器。

  ​	counting方法返回的收集器仅用作下游收集器。通过在 Stream上的 count方法，直接就有相同的功能，**因此压根没有理由使用collect(counting())**。这个属性还有15种Collectors方法。其中包含9种方法其名称以 summing、 averaging和 summarizing开头(相应的 Stream基本类型上就有相同的功能)。它们还包括 reducing、filtering、mapping、flatMapping和colectingAndThen方法。大多数程序员都能安全地避开这里的大多数方法。从设计的角度来看，这些收集器试图部分复制收集器中Stream的功能，以便下游收集器可以成为“ ministream”。

  ​	目前已经提到了3个Collectors方法。虽然它们都在Collectors中，但是并不包含集合。前两个是minBy和 maxBy，它们有一个比较器，并返回由比较器确定的Stream中的最少元素或者最多元素。它们是Stream接口中min和max方法的粗略概括，也是BinaryOperator中同名方法返回的二进制操作符，与收集器相类似。回顾一下在最畅销唱片范例中用过的BinaryOperator. maxBy方法。

  ​	最后一个Collectors方法是joining，它只在 CharSequence实例的 Stream中操作，例如字符串。它以参数的形式返回一个简单地合并元素的收集器。其中一种参数形式带有一个名为 delimiter(分界符)的 CharSequence参数，它返回一个连接 Stream元素并在相邻元素之间插入分隔符的收集器。如果传入一个逗号作为分隔符，收集器就会返回一个用逗号隔开的值字符串(但要注意，如果 Stream中的任何元素中包含逗号，这个字符串就会引起歧义)。这三种参数形式，除了分隔符之外，还有一个前缀和一个后缀。最终的收集器生成的字符串，会像在打印集合时所得到的那样，如[came，saw， conquered]。

---

+ 小结

  ​	总而言之，编写 Stream pipeline的本质是无副作用的函数对象。这适用于传人 Stream及相关对象的所有函数对象。终止操作中的 forEach应该只用来报告由 Stream执行的计算结果，而不是让它执行计算。为了正确地使用 Stream，必须了解收集器。最重要的收集器工厂是 toList、 toSet、 toMap、 groupingBy和 joining。

## E47 Stream要优先用Collection作为返回类型

+ 概述

  ​	许多方法都返回元素的序列。在Java8之前，这类方法明显的返回类型是集合接口Collection、Set和List；Iterable；以及数组类型。一般来说，很容易确定要返回这其中哪一种类型。标准是一个集合接口。如果某个方法只为for-each循环或者返回序列而存在，无法用它来实现一些Collection方法(一般是contains(Object))，那么就用Iterable接口吧。如果返回的元素是基本类型值，或者有严格的性能要求，就使用数组。在Java8中增加了 Stream，本质上导致给序列化返回的方法选择适当返回类型的任务变得更复杂了。

  ​		或许你曾听说过，现在 Stream是返回元素序列最明显的选择了，但如第45条所述，Stream并没有淘汰迭代：要编写出优秀的代码必须巧妙地将 Stream与迭代结合起来使用。如果一个APⅠ只返回一个 Stream，那些想要用for-each循环遍历返回序列的用户肯定要失望了。因为 Stream接口只在 Iterable接口中包含了唯一一个抽象方法， Stream对于该方法的规范也适用于 Iterable的。唯一可以让程序员避免用for-each循环遍历 Stream的是 Stream无法扩展 Iterable接口。

  ​	遗憾的是，这个问题还没有适当的解决办法。乍看之下，好像给 Stream的 iterator方法传入一个方法引用可以解决。这样得到的代码可能有点杂乱、不清晰，但也不算难以理解：

  ```java
  // Won't compile, due to limitations on Java's type inference
  for(ProcessHandle ph : ProcessHandle.allProcesses()::iterator) {
    // Process the process
  }
  ```

  ​	遗憾的是，如果想要编译这段代码，就会得到一条报错的信息：

  ```shell
  Test.java:6: error: method reference not expected here for (ProcessHandle ph : ProcessHandle.allProcesses()::iterator) {
  ^
  ```

  ​	为了使代码能够进行编译，必须将方法引用成适当参数化的Iterable：

  ```java
  // Hideous workaround to iterate over a stream
  for (ProcessHandle ph : (Iterable<ProcessHandle>) ProcessHandle.allProcesses()::iterator)
  ```

  ​	这个客户端代码可行，但是实际使用时过于杂乱、不清晰。更好的解决办法是使用适配器方法。JDK没有提供这样的方法，但是编写起来很容易，使用在上述代码中内嵌的相同方法即可。注意，在适配器方法中没有必要进行转换，因为Java的类型引用在这里正好派上了用场：

  ```java
  // Adapter from Stream<E> to Iterable<E>
  public static <E> Iterable<E> iterableOf(Stream<E> stream) {
    return stream::iterator;
  }
  ```

  ​	有了这个适配器，就可以利用for-each语句遍历任何Stream：

  ```java
  for(ProcessHandle p : iterableOf(ProcessHandle.allProcesses())) {
    // Process the process
  }
  ```

  ​	注意，（E34）中 Anagrams程序的 Stream版本是使用Files.lines方法读取词典，而迭代版本则使用了扫描器(scanner)。 Files.lines方法优于扫描器，因为后者默默地吞掉了在读取文件过程中遇到的所有异常。最理想的方式是在迭代版本中也使用Files.lines。这是程序员在特定情况下所做的一种妥协，比如当API只有 Stream能访问序列，而他们想通过for-each语句遍历该序列的时候。

  ​	反过来说，想要利用 Stream pipeline处理序列的程序员，也会被只提供Iterable的API搞得束手无策。同样地，JDK没有提供适配器，但是编写起来也很容易:

  ```java
  // Adapter from Iterable<E> to Stream<E>
  public static <E> Stream<E> streamOf(Iterable<E> iterable) {
    return StreamSupport.stream(iterable.spliterator(), false);
  }
  ```

  ​	如果在编写一个返回对象序列的方法时，就知道它只在 Stream pipeline中使用，当然就可以放心地返回Stream了。同样地，当返回序列的方法只在迭代中使用时，则应该返回Iterable。但如果是用公共的API返回序列，则应该为那些想要编写 Stream pipeline，以及想要编写for-each语句的用户分别提供，除非有足够的理由相信大多数用户都想要使用相同的机制。

  ​	Collection接口是Iterable的一个子类型，它有一个stream方法，因此提供了迭代和stream访问。**对于公共的、返回序列的方法，Collection或者适当的子类型通常是最佳的返回类型**。数组也通过 Arrays.asList和 Stream.of方法提供了简单的迭代和 stream访问。如果返回的序列足够小，容易存储，或许最好返回标准的集合实现，如ArrayList或者HashSet。但是**千万别在内存中保存巨大的序列，将它作为集合返回即可**。

  ​	如果返回的序列很大，但是能被准确表述，可以考虑实现一个专用的集合。假设想要返回一个指定集合的幂集（power set），其中包括它所有的子集。{a，b，c}的幂集是{{}，(a}，{b}，{c}，{a，b}，{a，c}，{b，c}，(a，b，c)}。如果集合中有n个元素，它的幂集就有2n个。因此，不必考虑将幂集保存在标准的集合实现中。但是，有了AbstractList的协助，为此实现定制集合就很容易了。

  ​	技巧在于，用幂集中每个元素的索引作为位向量，在索引中排第n位，表示源集合中第n位元素存在或者不存在。实质上，在二进制数0至2n-1和有n位元素的集合的幂集之间，有一个自然映射。代码如下:

  ```java
  // Return the power set of an input set as custom collection
  public class PowerSet {
    public static final <E> Collection<Set<E>> of(Set<E> s) {
      List<E> src = new ArrayList<>(s);
      if(src.size() > 30)
        throw new IllegalArgumentException("Set too big" + s);
      return new AbstractList<Set<E>>() {
        @Override public int size() {
          return 1 << src.size(); // 2 to the power srcSize
        }
        @Override public boolean contains(Object o) {
          return o instanceof Set && src.containAll((Set) o);
        }
        @Override public Set<E> get(int index) {
          Set<E> result = new HashSet<>();
          for(int i = 0; index != 0; i++, index >>= 1)
            if((index & 1) == 1)
              result.add(src.get(i));
          return result;
        }
      }
    }
  }
  ```

  ​	注意，如果输入值集合中超过30个元素， PowerSet.of会抛出异常。这正是用 Collection而不是用Stream或Iterable作为返回类型的缺点：Collection有一个返回int类型的size方法，它限制返回的序列长度为 Integer. MAX_VALUE或者2<sup>31</sup>-1。如果集合更大，甚至无限大，Collection规范确实允许size方法返回2<sup>31</sup>-1，但这并非是最令人满意的解决方案。

  ​	为了在 AbstractCollection上编写一个Collection实现，除了Iterable必需的那一个方法之外，只需要再实现两个方法：contains和size。这些方法经常很容易编写出高效的实现。如果不可行，或许是因为没有在迭代发生之前先确定序列的内容，返回 Stream或者 Iterable，感觉哪一种更自然即可。如果能选择，可以尝试着分别用两个方法返回。

  ​	**有时候在选择返回类型时，只需要看是否易于实现即可**。例如，要编写一个方法，用它返回一个输入列表的所有(相邻的)子列表。它只用三行代码来生成这些子列表，并将它们放在一个标准的集合中，但存放这个集合所需的内存是源列表大小的平方。这虽然没有幂集那么糟糕，但显然也是无法接受的。像给幂集实现定制的集合那样，确实很烦琐，这个可能还更甚，因为JDK没有提供基本的 Iterator实现来支持。

  ​	但是，实现输入列表的所有子列表的 Stream是很简单的，尽管它确实需要有点洞察力。我们把包含列表第一个元素的子列表称作列表的前缀。例如，(a，b，c)的前缀就是(a)、(a，b)和(a，b，c)。同样地，把包含最后一个元素的子列表称作后缀，因此(a，b，c)的后缀就是(a，b，c)、(b，c)和(c)。考验洞察力的是，列表的子列表不过是前缀的后缀(或者说后缀的前缀)和空列表。这一发现直接带来了一个清晰且相当简洁的实现：

  ```java
  // Returns a stream of all the sublists of its input list
  public class SubLists {
    public static <E> Stream<List<E>> of(List<E> list) {
      return Stream.concat(Stream.of(Collections.emptyList()),
                          prefixes(list).flatMap(SubLists::suffixes));
    }
    private static <E> Stream<List<E>> prefixes(List<E> list) {
      return IntStream.rangeClosed(1, list.size())
        .mapToObj(end -> list.subList(0, end));
    }
    private static <E> Stream<List<E>> suffixes(List<E> list) {
      return IntStream.range(0, list.size())
        .mapToObj(start -> list.subList(start, list.size()));
    }
  }
  ```

  ​	注意，它用 Stream.concat方法将空列表添加到返回的 Stream。另外还用 flatMap方法（E45）生成了一个包含了所有前缀的所有后缀的Stream。最后，通过映射IntStream.range和IntStream.rangeClosed返回的连续int值的 Stream，生成了前缀和后缀。通俗地讲，这一术语的意思就是指数为整数的标准for循环的 Stream版本。因此，这个子列表实现本质上与明显的嵌套式for循环相类似：

  ```java
  for (int start = 0; start < src.size(); start++)
    for (int end = start + 1; end <= src.size(); end++)
      System.out.println(src.subList(start, end));
  ```

  ​	这个for循环也可以直接翻译成一个 Stream。这样得到的结果比前一个实现更加简洁，但是可读性稍微差了一点。它本质上与（E45）中笛卡尔积的 Stream代码相类似：

  ```java
  // Returns a stream of all the sublists of its input list 
  public static <E> Stream<List<E>> of(List<E> list) {
    return IntStream.range(0, list.size())
      .mapToObj(start ->
               IntStream.rangeClosed(start + 1, list.size())
               .mapToObj(end -> list.subList(start, end)))
      .flatMap(x -> x);
  }
  ```

  ​	像前面的for循环一样，这段代码也没有发出空列表。为了修正这个错误，也应该使用concat，如前一个版本中那样，或者用rangeClosed调用中的(int)Math.signum(start)代替1。

  ​	子列表的这些 Stream实现都很好，但这两者都需要用户在任何更适合迭代的地方，采用stream-to-Iterable适配器，或者用Stream。Stream-to-Iterable适配器不仅打乱了客户端代码，在作者的机器上循环的速度还降低了2.3倍。专门构建的Collection实现(此处没有展示)相当烦琐，但是运行速度在作者的机器上比基于 Stream的实现快了约1.4倍。

---

+ 小结

  ​	总而言之，<u>在编写返回一系列元素的方法时，要记住有些用户可能想要当作 Stream处理，而其他用户可能想要使用迭代。要尽量两边兼顾</u>。

  ​	如果可以返回集合，就返回集合。如果集合中已经有元素，或者序列中的元素数量很少，足以创建一个新的集合，那么就返回个标准的集合，如 ArrayList。否则，就要考虑实现一个定制的集合，如幂集(power set)范例中所示。如果无法返回集合，就返回 Stream或者 Iterable，感觉哪一种更自然即可。如果在未来的Java发行版本中， Stream接口声明被修改成扩展了Iterable接口，就可以放心地返回 Stream了，因为它们允许进行 Stream处理和迭代。

## E48 谨慎使用Stream并行

P182
