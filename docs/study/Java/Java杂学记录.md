# Java杂学记录

## 1. GC

> [记一次线上频繁GC的排查过程](https://blog.csdn.net/weixin_42392874/article/details/89483496)
>
> [[JVM\]一次线上频繁GC的问题解决](https://www.cnblogs.com/zhengwangzw/p/10493562.html)
>
> [内存频繁GC问题查找分享](https://blog.csdn.net/u011983389/article/details/11126963)
>
> [Java 如何排查堆外内存使用情况？](https://www.v2ex.com/t/618931)
>
> [Java 对象内存布局](https://www.v2ex.com/t/356007)
>
> [Java 系统内存泄漏定位](https://www.v2ex.com/t/607922)
>
> [[分享] Java 对象内存布局](https://www.v2ex.com/t/356007)
>
> [Java 系统内存泄漏定位](https://www.v2ex.com/t/607922)
>
> [java内存dump文件导出与查看](https://blog.csdn.net/lsh2366254/article/details/84911374)
>
> [记一次频繁gc排查过程](https://www.jianshu.com/p/13915f4a562b)
>
> [[jvm\][面试]JVM 调优总结](https://www.cnblogs.com/diegodu/p/9849611.html)
>
> [深入浅出JVM调优，看完你就懂](https://blog.csdn.net/Javazhoumou/article/details/99298624)
>
> [static类/变量会不会被GC回收？](https://bbs.csdn.net/topics/80471342)

### 1. static的GC问题

>  [static类/变量会不会被GC回收？](https://bbs.csdn.net/topics/80471342)	

​	static变量一直都有指向，比如static int a = 999;那么一直有个a 引用 数值999，这里的a和数值999所占用的内存不会被回收。

​	如果static int[] a = new int[1024];然后之后又有 a = null;的操作，那么这个a所指向的int[1024]是以后会被GC的。

## 2. 动态代理

> spring默认使用jdk动态代理，如果类没有接口，则使用cglib。
>
> Spring5.X还是默认使用JDK动态代理，而SpringBoot2.X为了避免用户@Autowired使用不规范导致JDK动态代理失败，默认使用cglib
>
> [Java两种动态代理JDK动态代理和CGLIB动态代理](https://blog.csdn.net/flyfeifei66/article/details/81481222)
>
> [JDK和CGLIB动态代理区别](https://blog.csdn.net/yhl_jxy/article/details/80635012)
>
> [CGLib动态代理](https://www.cnblogs.com/wyq1995/p/10945034.html)
>
> [CGLIB(Code Generation Library) 介绍与原理](https://www.runoob.com/w3cnote/cglibcode-generation-library-intro.html)
>
> [ASM](https://www.jianshu.com/p/a1e6b3abd789)
>
> [秒懂Java动态编程（Javassist研究）](https://blog.csdn.net/ShuSheng0007/article/details/81269295)
>
> [什么鬼？弃用JDK动态代理，Spring5 默认使用 CGLIB 了？](https://blog.csdn.net/weixin_43167418/article/details/103900670)
>
> [JDK动态代理](https://www.cnblogs.com/zuidongfeng/p/8735241.html)

@Autowired

UserService userService； // JDK or CgLib都ok

@Autowired

UserServiceImpl userService; // JDK报错，因为该类型不是接口，JDK是代理是创建了接口类的一个实现类。CgLib则是根据继承实现的。

## 3. final

> 性能影响：总之就是影响甚微，没必要通过考虑使用final来"改变性能"。
>
> final只作用于编译阶段，运行阶段根本不存在final关键字更别说什么影响了，顶多就是编译过程中不同虚拟机对含有final的代码的解释方式不同，进而可能有不同的执行逻辑。但本质上，final在运行时没有作为。
>
> [Final关键字之于Java性能的影响](https://www.jianshu.com/p/e50029ec0ea7)
>
> [java方法中变量用final修饰对性能有影响！你觉得呢？](https://zhidao.baidu.com/question/459147671.html)
>
> [为什么一些人喜欢在java代码中能加final的变量都加上final](https://blog.csdn.net/qq_31433709/article/details/87823478)
>
> 

​	如果试图通过多用final来提升性能啥的，那还是算了。提升性能应该在算法使用、设计模式、业务逻辑等方面下功夫，而不是咬文嚼字地钻牛角尖。

## 4. 并发

> [Java并发——Executor框架详解（Executor框架结构与框架成员）](https://blog.csdn.net/tongdanping/article/details/79604637)
>
> [java并发编程：Executor、Executors、ExecutorService](https://blog.csdn.net/weixin_40304387/article/details/80508236)
>
> 锁
>
> [使用synchronized同步对象却有多个线程能同时访问，使用lock锁却达到目的了，不知道为什么求大神回](https://bbs.csdn.net/topics/391087139?page=1)

## 5. 布隆过滤器

> [布隆过滤器介绍和应用](https://www.jianshu.com/p/03c8dad08035)

## 6. 内存分析

> [性能优化工具（八）-MAT](https://www.jianshu.com/p/97251691af88)
>
> [Java内存分析工具--IDEA的JProfiler和JMeter插件](https://blog.csdn.net/qq_19674905/article/details/80824858)
>
> [MAT使用笔记](https://blog.csdn.net/zgmzyr/article/details/8232323)

## 7. 浅拷贝和深拷贝

> [Java中的clone方法-理解浅拷贝和深拷贝](https://www.cnblogs.com/JamesWang1993/p/8526104.html)

## 8. 类型擦除

> [Java类型擦除机制](https://www.cnblogs.com/chenpi/p/5508177.html)
>
> 范型仅在编译时有效，运行时都是Object。范型不具备继承关系。
>
> 通配符"?" => 编译时不知道类型，所以只能get不能add；
>
> 有界通配符"\<? extends Object\>","\<? super Object\>"可以add对应Object或其子类/父类。（因为我们编译前给定了父类/子类）

## 9. 编码问题

> [几种常见的编码格式 ](https://www.cnblogs.com/mlan/p/7823375.html)
>
> [Java中char到底是多少字节？](https://www.iteye.com/topic/47740)

