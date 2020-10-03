# Spring杂记

> [Spring常见面试题总结（超详细回答）](https://blog.csdn.net/a745233700/article/details/80959716)
>
> [Spring IOC介绍与4种注入方式](https://zhuanlan.zhihu.com/p/34405799)
>
> [深入解析spring中用到的九种设计模式](https://kaiwu.lagou.com/java_architect.html)
>
> [Spring系列之beanFactory与ApplicationContext](https://www.cnblogs.com/xiaoxi/p/5846416.html)
>
> [Spring-bean的循环依赖以及解决方式](https://blog.csdn.net/u010853261/article/details/77940767)
>
> Spring的循环依赖的理论依据其实是基于Java的引用传递，当我们获取到对象的引用时，对象的field或则属性是可以延后设置的(但是构造器必须是在获取引用之前)。
>
> [为什么使用Spring的@autowired注解后就不用写setter了？](https://blog.csdn.net/qq_19782019/article/details/85038081)
>
> Spring偷偷的把我们用@autowired标记过的属性的【访问控制检查】给关闭了，即对每个属性进行了【setAccessible（true）】的设置，导致这些属性即使被我们标记了【private】,Spring却任然能够不通过getter和setter方法来访问这些属性,达到一定的目的。
>
> [Spring Bean 的scope什么时候设置为prototype，什么时候设置为singleton](https://blog.csdn.net/q276513307/article/details/78393599)
>
> 1.对于有实例变量的类，要设置成prototype；没有实例变量的类，就用默认的singleton 
> 2.Action一般我们都会设置成prototype，而Service只用singleton就可以。
>
> [spring boot 使用ThreadLocal实例](https://blog.csdn.net/qq_27127145/article/details/83894400)

