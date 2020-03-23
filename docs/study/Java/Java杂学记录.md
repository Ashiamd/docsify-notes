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
> [Java两种动态代理JDK动态代理和CGLIB动态代理](https://blog.csdn.net/flyfeifei66/article/details/81481222)
>
> 
>
> [CGLIB(Code Generation Library) 介绍与原理](https://www.runoob.com/w3cnote/cglibcode-generation-library-intro.html)
>
> [ASM](https://www.jianshu.com/p/a1e6b3abd789)
>
> [秒懂Java动态编程（Javassist研究）](https://blog.csdn.net/ShuSheng0007/article/details/81269295)

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
> s
>
> 

​	如果试图通过多用final来提升性能啥的，那还是算了。提升性能应该在算法使用、设计模式、业务逻辑等方面下功夫，而不是咬文嚼字地钻牛角尖。