# 《Effective-Java第三版》-读书笔记-03

> + `1、`、`2、`表示：第1章、第2章
>
> + `E1`、`E2`表示：第1条、第2条
> + `* X、`表示：个人认为第X章是重点（注意点）
> + `* EX`表示：个人认为第X条是重点（注意点）
>
> 全书共12章，90条目
>
> 下面提到的“设计模式”，指《Design Patterns - Elements of Reusable Object-Oriented Software》一书中提到的23种设计模式

# 9、通用编程
​	本章主要讨论Java语言的细枝末节，包含局部变量的处理、控制结构、类库的用法、各种数据类型的用法，以及两种不是由语言本身提供的机制(**反射机制和本地方法**)的用法。最后讨论了优化和命名惯例。
## E57 将局部变量的作用域最小化
- 概述

  ​	本条目与（E15）本质上是类似的。将局部变量的作用域最小化，可以增强代码的可读性和可维护性，并降低出错的可能性。

  ​	<u>较早的编程语言(如C语言)要求局部变量必须在代码块的开头进行声明</u>，出于习惯，有些程序员目前还是继续这样做。这个习惯应该改正。在此提醒，Java允许你在任何可以出现语句的地方声明变量。

  ​	**要使局部变量的作用域最小化，最有力的方法就是在第一次要使用它的地方进行声明**。如果变量在使用之前进行声明，这只会造成混乱—对于试图理解程序功能的读者来说，这又多了一种只会分散他们注意力的因素。等要用到该变量时，读者可能已经记不起该变量的类型或者初始值了。

  ​	过早地声明局部变量不仅会使它的作用域过早地扩展，而且结束得过晚。局部变量的作用域从它被声明的点开始扩展，一直到外围块的结束处。如果变量是在“使用它的块”之外被声明的，当程序退出该块之后，该变量仍是可见的。如果变量在它的目标使用区域之前或者之后被意外地使用，后果将可能是灾难性的。

  ​	**几乎每一个局部变量的声明都应该包含一个初始化表达式**。如果你还没有足够的信息来对一个变量进行有意义的初始化，就应该推迟这个声明，直到可以初始化为止。<u>这条规则有个例外的情况与try-catch语句有关</u>。

  + **如果一个变量被一个方法初始化，而这个方法可能会抛出一个受检异常，该变量就必须在try块的内部被初始化**。
  + **如果变量的值必须在try块的外部用到，它就必须在try块之前被声明，但是在try块之前，它还不能被“有意义地初始化”**。请参照（E65）中的例子。

  ​	循环中提供了特殊的机会来将变量的作用域最小化。无论是传统的for循环，还是for-each形式的for循环，都允许声明循环变量(loop variable)，它们的作用域被限定在正好需要的范围之内。(这个范围包括循环体，以及循环体之前的初始化、测试、更新部分。因此，如果在循环终止之后不再需要循环变量的内容，for循环就优先于whie循环。

  ​	例如，下面是一种遍历集合的首选做法（E58）

  ```java
  // Preferred idiom for iterating over a collection or array
  for (Element e : c) {
    ... // Do Something with e
  }
  ```

  ​	如果需要访问迭代器，可能要调用它的remove方法，首先做法是利用传统的for循环代替for-each循环：

  ```java
  // Idiom for iterating when you need the iterator
  for (Iterator<Element> i = c.iterator(); i.hasNext(); ) {
    Element e = i.next();
    ... // Do something with e and i
  }
  ```

  ​	为了弄清楚为什么这些for循环比while循环更好，请参考下面的代码片段，它包含两个while循环，以及一个Bug：

  ```java
  Iterator<Element> i = c.iterator();
  while(i.hasNext()) {
    doSomething(i.next());
  }
  ...
  Iterator<Element> i2 = c2.iterator();
  while(i.hasNext()) { // BUG!
    doSomethingElse(i2.next());
  }
  ```

  ​	第二个循环中包含一个“剪切一粘贴”错误:本来是要初始化一个新的循环变量i2，却使用了旧的循环变量i，遗憾的是，这时i仍然还在有效范围之内。结果代码仍然可以通过编译，运行的时候也不会抛出异常，但是它所做的事情却是错误的。第二个循环并没有在c2上迭代，而是立即终止，造成c2为空的假象。因为这个程序的错误是悄然发生的，所以可能在很长时间内都不会被发现。

  ​	如果类似的“剪切-粘贴”错误发生在前面任何一种for循环中，结果代码根本就不能通过编译。在第二个循环开始之前，第一个循环的元素(或者迭代器)变量已经不在它的作用域范围之内了。下面就是一个传统for循环的例子：

  ```java
  for (Iterator<Element> i = c.iterator(); i.hasNext(); ){
    Element e = i.next();
    ... // Do something with e and i
  }
  ...
  // Compile-time error - cannot find symbol i
  for(Iterator<Element> i2 = c2.iterator(); i.hasNext(); ) {
    Element e2 = i2.next();
    ... // Do something with e2 and i2
  }
  ```

  ​	**如果使用for循环，犯这种“剪切-粘贴”错误的可能性就会大大降低，因为通常没有必要在两个循环中使用不同的变量名**。循环是完全独立的，所以重用元素(或者迭代器)变量的名称不会有任何危害。实际上，这也是很流行的做法。

  ​	使用fαr循环与使用whie循环相比还有另外一个优势：更简短，从而增强了可读性。

  ​	下面是另外一种对局部变量的作用域进行最小化的循环做法：

  ```java
  for (int i = 0, n = expensiveComputation(); i < n; i++) {
    ... // Do something with i;
  }
  ```

  ​	关于这种做法要关注的重点是，它具有两个循环变量i和n，二者具有完全相同的作用域。第二个变量n被用来保存第一个变量的极限值，从而避免在每次迭代中执行冗余计算。通常，如果循环测试中涉及方法调用，并且可以保证在每次迭代中都会返回同样的结果，就应该使用这种做法。

  ​	**最后一种“将局部变量的作用域最小化”的方法是使方法小而集中**。如果把两个操作(activity)合并到同一个方法中，与其中一个操作相关的局部变量就有可能会出现在执行另个操作的代码范围之内。为了防止这种情况发生，只需将这个方法分成两个：每个操作用个方法来完成。

## E58 for-each 循环优先于传统的for循环

+ 概述

  ​	如（E45）所述，有些任务最好结合 Stream来完成，有些最好结合迭代完成。下面是用个传统的for循环遍历集合的例子：

  ```java
  // Not the best way to iterate over a collection!
  for (Iterator<Element> i = c.iterator(); i.hasNext(); ){
  	Element e = i.next();
    ... // Do something with e
  }
  ```

  ​	用传统的for循环遍历数组的做法如下：

  ```java
  // Not the best way to iterate over an array!
  for (int i = 0; i < a.length; i++) {
    ... // Do something with a[i]
  }
  ```

  ​	这些做法都比while循环(详见第57条)更好，但是它们并不完美。迭代器和索引变量都会造成一些混乱——而你需要的只是元素而已。而且，它们也代表着出错的可能。迭代器在每个循环中出现三次，索引变量在每个循环中出现四次，其中有两次让你很容易出错。旦出错，就无法保证编译器能够发现错误。最后一点是，这两个循环是截然不同的，容器的类型转移了不必要的注意力，并且为修改该类型增加了一些困难。

  ​	for-each循环(官方称之为“增强的for语句”)解决了所有问题。通过完全隐藏迭代器或者索引变量，避免了混乱和出错的可能。这种模式同样适用于集合和数组，同时简化了将容器的实现类型从一种转换到另一种的过程：

  ```java
  // The preferred idiom for iterating over collections and arrays
  for (Element e : elements) {
    ... // Do something with e
  }
  ```

  ​	当见到冒号(:)时，可以把它读作“在……里面”。因此上面的循环可以读作“对于元素elements中的每一个元素e”。注意，利用 for-each循环不会有性能损失，甚至用于数组也样：它们产生的代码本质上与手工编写的一样。

  ​	对于嵌套式迭代，for-each循环相对于传统for循环的优势还会更加明显。下面就是人们在试图对两个集合进行嵌套迭代时经常会犯的错误：

  ```java
  // Can you spot the bug?
  enum Suit { CLUB, DIAMOND, HEART, SPADE }
  enum Rank { ACE, DUECE, THREE, FOUR, FIVE, SIX, SEVEN, EIGHT, NIGHT, TEN, JACK, QUEEN, KING }
  ...
  static Collection<Suit> suits = Arrays.asList(Suit.values());
  static Collection<Rank> ranks = Arrays.asList(Rank.values());
  
  List<Card> deck = new ArrayList<>();
  for (Iterator<Suit> i = suits.iterator(); i.hasNext(); )
    for(Iterator<Rank> j = ranks.iterator(); j.hasNext(); )
      deck.add(new Card(i.next(), j.next()));
  ```

  ​	如果之前没有发现这个Bug也不必难过。许多专家级的程序员偶尔也会犯这样的错误。问题在于，在迭代器上对外部的集合(suits)调用了太多次next方法。它应该从外部的循环进行调用，以便每种花色调用一次，但它却是从内部循环调用，因此每张牌调用一次。在用完所有花色之后，循环就会抛出 NoSuchElementException异常。

  ​	如果真的那么不幸，并且外部集合的大小是内部集合大小的几倍(可能因为它们是相同的集合)，循环就会正常终止，但是不会完成你想要的工作。例如，下面就是一个考虑不周的尝试，想要打印一对骰子的所有可能的滚法：

  ```java
  // Same bug, different symptom!
  enum Face { ONE, TWO, THREE, FOUR, FIVE, SIX }
  ...
  Collection<Face> faces = EnumSet.allOf(Face.class);
  
  for(Iterator<Face> i = faces.iterator(); i.hasNext(); )
    for(Iterator<Face> j = faces.iterator(); j.hasNext(); )
      System.out.println(i.next() + " " + j.next());
  ```

  ​	这个程序不会抛出异常，而是只打印6个重复的词(从“ ONE ONE”到“ SIX SIX”)，而不是预计的那36种组合。

  ​	为了修正这些示例中的Bug，必须在外部循环的作用域中添加一个变量来保存外部元素:

  ```java
  // Fixed, but ugly - you can do better!
  for(Iterator<Suit> i = suits.iterator(); i.hasNext(); ) {
    Suit suit = i.next();
    for(Iterator<Rank> j = ranks.iterator(); j.hasNext(); )
      deck.add(new Card(suit, j.next()));
  }
  ```

  ​	如果使用的是嵌套式 for-each循环，这个问题就会完全消失。产生的代码将如你所希望的那样简洁:

  ```java
  // Preferred idiom for nested iteration on collections and arrays
  for(Suit suit : suits)
    for(Rank rank : ranks)
      deck.add(new Card(suit, rank));
  ```

  ​	遗憾的是，有<u>三种常见的情况无法使用for-each循环</u>：

  + 解构过滤——如果需要遍历集合，并删除选定的元素，就需要使用显式的迭代器，以便可以调用它的 remove方法。**使用Java8中增加的Collection的removeIf方法，常常可以避免显式的遍历**。
  + 转换——如果需要遍历列表或者数组，并取代它的部分或者全部元素值，就需要列表迭代器或者数组索引，以便设定元素的值。
  + 平行迭代——如果需要并行地遍历多个集合，就需要显式地控制迭代器或者索引变量，以便所有迭代器或者索引变量都可以同步前进(就如上述有问题的牌和骰子的示例中无意间所示范的那样)。

  ​	如果你发现自己处于以上任何一种情况之下，就要使用普通的for循环，并且要警惕本条目中提到的陷阱。**for-each循环不仅能遍历集合和数组，还能遍历实现Iterable接口的任何对象**，该接口中只包含单个方法，具体如下：

  ```java
  public interface Iterable<E> {
    // Returns an iterator over the elements in this iterable
    Iterator<E> iterator();
  }
  ```

  ​	**如果不得不从头开始编写自己的Iterator实现，其中还是有些技巧的，但是如果编写的是表示一组元素的类型，则应该坚决考虑让它实现Iterable接口，甚至可以选择让它不要实现Collection接口**。这样，你的用户就可以利用for-each循环遍历类型，他们会永远心怀感激的。

---

+ 小结

  ​	总而言之，与传统的for循环相比，for-each循环在简洁性、灵活性以及出错预防性方面都占有绝对优势，并且没有性能惩罚的问题。因此，当可以选择的时候，for-each循环应该优先于for循环。

## E59 了解和使用类库

P217

