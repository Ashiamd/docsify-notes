# Spring学习杂记

## 1. 依赖注入

> [Spring学习（十八）Bean 的三种依赖注入方式介绍](https://www.cnblogs.com/lirunzhou/p/9843176.html)
>
> [@Autowired的使用：推荐对构造函数进行注释](https://blog.csdn.net/qq_22873427/article/details/73718952)

# 2. AOP

> [Aspectj与Spring AOP比较 - 简书 (jianshu.com)](https://www.jianshu.com/p/872d3dbdc2ca) => 图文并茂，推荐阅读

# 3. Spring事务

> + [Spring中同一类@Transactional修饰方法相互调用的坑_前路无畏的博客-CSDN博客](https://blog.csdn.net/fsjwin/article/details/109211355)	<=	避坑推荐
> + [Spring事务-随笔-Ashiamd - 博客园 (cnblogs.com)](https://www.cnblogs.com/Ashiamd/p/15085827.html)
> + [Spring事务测试github项目](https://github.com/Ashiamd/SpringTransactionTest)

1. @Transactional 由于serviceImp实现的service，所以AOP默认用的Spring AOP中的jdk动态代理。因此private、protected、包级、static的不能生效，但是不报错。另外由于AOP，同类下的其他方法上的@Transactional不生效，因为是类内部方法调用，动态代理不生效。解决方法：1、写在不同的类里；2、同类，但是用AspectJ获取代理对象，用代理对象再调用同类的B方法。
(ps: 可以在 `org.springframework.transaction.interceptor.TransactionAspectSupport#invokeWithinTransaction` 中打断点查看一些事务调用情况)

2. @Transactional指定的spring事务传播对 TransactionTemplate transactionTemplate 有效，TransactionTemplate transactionTemplate 就是单纯开启事务的，就好像原本的@Transactional在方法前后通过AOP来开启、提交事务，我们这手动提交事务罢了。然后mysql事务本身和spring的事务传递是两个东西，所以transactionTemplate也受当前使用它的方法的spring事务传播级别的影响。

3. REQUIRES_NEW 和 NESTED的区别，前者开启和原事务完全无关的新事务，回滚是独立的新事务被回滚；后者如果已经存在事务，则仅设置savepoint，和原事务是同一个事务，不过就是后者回滚只回滚到自己的检查点。如果原本没有事务，那么NESTED就和REQUIRES一样。

4. 由于@Transactional这边Spring传播级别通过AOP实现，所以调用方法的时候就确认是否有父事务环境了，不会等运行到中间调用B之后又重新判断(Propagation.SUPPORTS就是一个典型例子)。

5. 同一个类中方法调用会可能导致@Transactional失效，重新使得@Transcational生效的方法：
    1. pom.xml 中添加AspectJ:
    ```xml
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>
    ```
    2. 启动类上添加 @EnableAspectJAutoProxy(exposeProxy = true)
    3. AopContext.currentProxy()操作当前的代理类
    ```java
    AopContext.currentProxy().A; // 调用代理类的.A方法。
    ```

6. TransactionTemplate相关

    + TransactionTemplate transactionTemplate的传播级别和当前方法上的@Transcational( propagation = Propagation.XXXX) 有关！！（比如A是NEVER，A通过transactionTemplate调用B，B是REQUIRED，那么transactionTemplate由于调用B之前先新建了事务，所以A抛异常，由于A直接抛异常，所以B压根没被执行）
    + transactionTemplate.execute本身就是开一个事务，但是同样要遵循该方法设定的spring事务传播级别。如果A的事务传播级别为Propagation.NEVER，那么A调用B，如果B新建事务（RQUIRED、REQUIRES_NEW），那么B执行完后，A直接抛出异常，这之前的修改都是commit到数据库的。但是B的操作由于是新的事务，只要B不抛出新异常，就仍然改库。
    + 理解Nested的关键是savepoint。它与PROPAGATION_REQUIRES_NEW的区别是，PROPAGATION_REQUIRES_NEW另起一个事务，将会与它的父事务相互独立，而Nested的事务和它的父事务是相依的，它的提交是要等和它的父事务一块提交的。也就是说，如果父事务最后回滚，它也要回滚的。而Nested事务的好处是它有一个savepoint。也就是说ServiceB.methodB失败回滚，那么ServiceA.methodA也会回滚到savepoint点上，ServiceA.methodA可以选择另外一个分支，比如ServiceC.methodC，继续执行，来尝试完成自己的事务。但是这个事务并没有在EJB标准中定义。
    
7. Spring AOP知识回顾

    1. 对于基于接口动态代理的AOP事务增强来说，由于接口的方法是public的，这就要求实现类的实现方法必须是public的， 不能是protected，private等，同时不能使用static的修饰符。

       所以，可以实施接口动态代理的方法只能是使用“public” 或 “public final”修饰符的方法，其它方法不可能被动态代理，相应的也就不能实施AOP增强，也即不能进行Spring事务增强。

    2. 基于CGLib字节码动态代理的方案是通过扩展被增强类，动态创建子类的方式进行AOP增强植入的。由于使用final、static、private修饰符的方法都不能被子类覆盖，相应的，这些方法将不能被实施的AOP增强。

    *ps：如果我们自己用jdk动态代理的原始写法，其实可以setAccessible(true)访问private修饰的东西*

> [spring事务传播级别](https://blog.csdn.net/qq_36094023/article/details/90544286)
> [Spring五个事务隔离级别和七个事务传播行为](https://www.cnblogs.com/wj0816/p/8474743.html)
> [transactionTemplate用法](https://blog.csdn.net/qq_20009015/article/details/84863295)
> [Spring中Transactional放在类级别和方法级别上有什么不同？](https://zhidao.baidu.com/question/1500582510886123139.html)
> [Spring @Transactional属性可以在私有方法上工作吗？](http://www.mianshigee.com/question/172320jjt/)
> [Aspectj与Spring AOP比较](https://www.jianshu.com/p/872d3dbdc2ca)
> [JDK动态代理](https://www.cnblogs.com/zuidongfeng/p/8735241.html)
> [Spring ： REQUIRED和NESTED的区别](https://blog.csdn.net/qq_31967241/article/details/107764496)
> [不同类的方法 事务问题_Spring 事务原理和使用，看完这一篇就足够了](https://blog.csdn.net/weixin_42367472/article/details/112636467)
> [Spring中同一类@Transactional修饰方法相互调用的坑](https://blog.csdn.net/fsjwin/article/details/109211355)
> [@Transactional同类方法调用不生效及解决方法](https://blog.csdn.net/weixin_38898423/article/details/113835501)
> [Spring中事务的Propagation（传播性）的取值](https://blog.csdn.net/zhang_shufeng/article/details/38706725)
> [spring+mybatis 手动开启和提交事务](https://www.cnblogs.com/xujishou/p/6210012.html)
> [AopContext.currentProxy()](https://blog.csdn.net/qq_29860591/article/details/108728150)

