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

P141

