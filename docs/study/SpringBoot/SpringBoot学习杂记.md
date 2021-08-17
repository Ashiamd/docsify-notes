# SpringBoot学习杂记

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

1. @Transactional 由于serviceImp实现的service，所以AOP默认用的Spring AOP中的jdk动态代理。因此private、protected、包级、static的不能生效，但是不报错。另外由于AOP，同类下的其他方法上的@Transactional不生效，因为是类内部方法调用，动态代理不生效。

  解决方法：

  1. 写在不同的类里；

     ```java
     // eg：
     @Service("studentService")
     public class StudentServiceImpl implements StudentService {
       
       @Resource
       private TeacherService teacherService;
       
       @Transactional(rollbackFor = Throwable.class, propagation = Propagation.REQUIRES_NEW)
       @Override
       public void A(Integer id) throws Exception {
     		this.insert(id);
         teacherService.insert(id);
       }
     }
     ```

  2. 同类，但是用AspectJ获取代理对象，用代理对象再调用同类的B方法；

     ```java
     // eg:
     @Service("studentService")
     public class StudentServiceImpl implements StudentService {
       
       @Transactional(rollbackFor = Throwable.class, propagation = Propagation.REQUIRES_NEW)
       @Override
       public void A(Integer id) throws Exception {
     		this.insert(id);
         ((StudentServiceImpl)AopContext.currentProxy()).B(id);
       }
     }
     ```

  3. 同类，依赖注入自己，再调用注入的对象的方法

     ```java
     // eg:
     @Service("studentService")
     public class StudentServiceImpl implements StudentService {
     
       @Resource
       private StudentService studentService;
     
       @Transactional(rollbackFor = Throwable.class, propagation = Propagation.REQUIRES_NEW)
       @Override
       public void A(Integer id) throws Exception {
         this.insert(id);
         studentService.insert(id);
       }
     }
     ```

  (ps: 可以在 `org.springframework.transaction.interceptor.TransactionAspectSupport#invokeWithinTransaction` 中打断点查看一些事务调用情况)

2. @Transactional指定的spring事务传播对 TransactionTemplate transactionTemplate 同样有效。

    + TransactionTemplate transactionTemplate可以通过`transactionTemplate.setPropagationBehavior(TransactionDefinition.PROPAGATION_NEVER);`设置事务传播级别。
    + 注意，如果在@Transactional方法内又使用transactionTemplate，那可能导致最后开了两个事务（具体看transactionTemplate设置的事务传播级别是什么，如果@Transactional和transactionTemplate都是用REQUIRES_NEW，那就是两个不相干的事务了）

3. REQUIRES_NEW 和 NESTED的区别，前者开启和原事务完全无关的新事务，回滚是独立的新事务被回滚；后者如果已经存在事务，则仅设置savepoint，和原事务是同一个事务，不过就是后者回滚只回滚到自己的检查点。如果原本没有事务，那么NESTED就和REQUIRES一样都是新建事务。

4. 由于@Transactional这边Spring传播级别通过AOP实现，所以调用方法的时候就确认是否有父事务环境了，不会等运行到中间调用B之后又重新判断(所以如果外层是Propagation.NEVER，即时方法内临时调用另一个事务方法，也不会抛出异常，外部方法始终无事务，内部被调用方法是独立的一个事务)。

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

    - transactionTemplate.execute本身就是开一个事务，也可以手动设定事务传播级别等。和@Transactional注解的区别就是注解的形式在方法执行前设置事务传播级别和开启事务，而transactionTemplate只在execute的时候才开启事务。

    + TransactionTemplate transactionTemplate可以通过`transactionTemplate.setPropagationBehavior(TransactionDefinition.PROPAGATION_NEVER);`设置事务传播级别

7. Spring AOP知识回顾

    1. 对于基于接口动态代理的AOP事务增强来说，由于接口的方法是public的，这就要求实现类的实现方法必须是public的， 不能是protected，private等，同时不能使用static的修饰符。

       所以，可以实施接口动态代理的方法只能是使用“public” 或 “public final”修饰符的方法，其它方法不可能被动态代理，相应的也就不能实施AOP增强，也即不能进行Spring事务增强。

    2. 基于CGLib字节码动态代理的方案是通过扩展被增强类，动态创建子类的方式进行AOP增强植入的。由于使用final、static、private修饰符的方法都不能被子类覆盖，相应的，这些方法将不能被实施的AOP增强。

    *ps：如果我们自己用jdk动态代理的原始写法，其实可以setAccessible(true)访问private修饰的东西*

> [spring事务传播级别](https://blog.csdn.net/qq_36094023/article/details/90544286)
> 
> [Spring五个事务隔离级别和七个事务传播行为](https://www.cnblogs.com/wj0816/p/8474743.html)
> 
> [transactionTemplate用法](https://blog.csdn.net/qq_20009015/article/details/84863295)
> 
> [Spring中Transactional放在类级别和方法级别上有什么不同？](https://zhidao.baidu.com/question/1500582510886123139.html)
> 
> [Spring @Transactional属性可以在私有方法上工作吗？](http://www.mianshigee.com/question/172320jjt/)
> 
> [Aspectj与Spring AOP比较](https://www.jianshu.com/p/872d3dbdc2ca)
> 
> [JDK动态代理](https://www.cnblogs.com/zuidongfeng/p/8735241.html)
> 
> [Spring ： REQUIRED和NESTED的区别](https://blog.csdn.net/qq_31967241/article/details/107764496)
> 
> [不同类的方法 事务问题_Spring 事务原理和使用，看完这一篇就足够了](https://blog.csdn.net/weixin_42367472/article/details/112636467)
> 
> [Spring中同一类@Transactional修饰方法相互调用的坑](https://blog.csdn.net/fsjwin/article/details/109211355)
> 
> [@Transactional同类方法调用不生效及解决方法](https://blog.csdn.net/weixin_38898423/article/details/113835501)
> 
> [Spring中事务的Propagation（传播性）的取值](https://blog.csdn.net/zhang_shufeng/article/details/38706725)
> 
> [spring+mybatis 手动开启和提交事务](https://www.cnblogs.com/xujishou/p/6210012.html)
> 
> [AopContext.currentProxy()](https://blog.csdn.net/qq_29860591/article/details/108728150)

# 4. 异常处理

> [Spring的@ExceptionHandler和@RestControllerAdvice使用-随笔 - Ashiamd - 博客园 (cnblogs.com)](https://www.cnblogs.com/Ashiamd/p/15045197.html)

- 如果项目中Controller继承某个带有@ExceptionHandler注解方法的类，那么Controller抛出异常时，会优先走该@ExceptionHandler注解的方法。
- 此时如果有另外带有@RestControllerAdvice注解的全局异常处理器，其只处理Controller继承的@ExceptionHandler范围外的异常。
- 如果@ExceptionHandler范围很大，比如是Throwable.class，那么所有异常只走Controller继承的异常处理方法，不会经过全局异常处理器。

（ps：@ExceptionHandler存在多个时，最具体的一个生效，比如Throwable.class和RuntimeException.class，如果抛出后者，则拦截后者的方法生效）

```java
// 举例子(下面共1. 2. 3. 三个类)：
/*

情况一：TestController抛出 ArrayStoreException，那么被ExceptionHandlers处理(如果ExceptinonHandlers再上抛，就直接服务停止了，而不是被ExceptionHandlers处理)

  情况二：TestController抛出 ArrayStoreException以外的异常（超出AbstractController拦截的范围），那么被ExceptionHandlers处理（同样如果再上抛，直接服务停止）

  情况三：假设AbstractController中的handleThrowable上的@ExceptionHandler(ArrayStoreException.class)改成@ExceptionHandler(Throwable.class)，那么所有异常只被AbstractController处理，全局异常处理器无作为（当然如果其他Controller没有继承AbstractController的话，就会抛异常被全局异常处理器ExceptionHandlers处理）。

*/

// 1. 带有@ExceptionHandler注解方法的 AbstractController
public abstract class AbstractController {
  @ExceptionHandler(ArrayStoreException.class)
  public Object handleThrowable(HttpServletRequest request, Throwable e) {}
}

// 2. 全局异常处理器
@RestControllerAdvice
public class ExceptionHandlers {
  @ExceptionHandler(Throwable.class)
  public String handleThrowable(HttpServletRequest request, Throwable e) {}
}

// 3. 普通Controller
@RestController
@RequestMapping("/test")
public class TestController extends AbstractController {
  @GetMapping("/test001")
  public String test001() throws Throwable {}
}
```

