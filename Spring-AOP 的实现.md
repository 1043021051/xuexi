说说 Spring-AOP 的实现
====
概述
----
#### OOP 和 AOP 的比较

OOP：面向对象编程，允许开发者定义纵向的关系，但并不适用于定义横向的关系，导致了大量代码的重复，不利于各个模块的重用。AOP：面向切面编程，作为面向对象的一种补充，用于将那些于业务无关，但却对多个对象产生影响的公共行为和逻辑，抽取并封装为一个可重用的模块，这个模块名为“切面”。这种模式减少了系统中的重复代码，降低了模块间的耦合度，同时提高了系统的可维护性，可用于权限认证，日志，事务处理等。

### AOP 的底层实现
AOP 实现的关键在于「代理模式」，AOP 主要分为静态代理和动态代理。静态代理的代表为 AspectJ；动态代理的代表为 Spring AOP。

所谓静态代理，就是 AOP 框架会在编译阶段生成 AOP 代理类，因此也称为编译时增强，它会在编译阶段将 AspectJ（切面）weaving（织入）到 Java 字节码中，运行的时候就是增强之后的 AOP 对象。所谓动态代理，就是 AOP 框架不会去修改字节码，而是每次运行时在内存中临时为方法生成一个 AOP 对象，这个 AOP 对象包含了目标对象的全部方法，并且在特定的切点做了增强处理，并调回原对象。
AOP 的相关概念 

Aspect（切面）：被抽取的公共模块，可能会横切为多个对象。在 Spring AOP 中，切面可以使用通用类（基于模式的风格）或者普通类中以 @AspectJ 注解来实现。Join Point（连接点）：指会被拦截的方法。一个被代理对象中有很多方法，一些方法会被拦截到，并为其做增强处理。Pointcut（切入点）：指被拦截到的方法。通过切入点表达式来指定要被拦截的方法。Advice（通知）：对切入点进行增强处理。通知有各种类型：

Before advice（前置通知）：在连接点未执行前执行的通知，这个通知不能阻止连接点的执行（除非有异常）After returning advice（返回后通知）：在连接点正常完成后执行的通知。After throwing advice（抛出异常后通知）：在连接点抛出异常退出时执行的通知。After (finally) advice（后通知）：在连接点退出的时候执行的通知。（不管是正常返回还是异常退出）Around Advice（环绕通知）：包围一个连接点的通知。
Introduction（引入）：被称为内部类型声明。声明额外的方法或者某个类型的字段。Spring 允许引入新的接口（以及一个对应的实现）到任何被代理的对象。例如，可以使用一个引入来使 bean 实现 IsModified 接口，以便简化缓存机制。Targt Objct（目标对象）：被一个或者多个切面所通知的对象，也称为被通知（被代理）对象。Weaving（织入）：把增强应用到目标对象从而创建新的代理对象的过程。Spring 是在运行时完成织入。

建立过程
-----
![Image text](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/58c56cf341d14a889d84e575dda25e34~tplv-k3u1fbpfcp-zoom-1.image)

#### 1. 设置切面
```
/**
 * 用于记录日志的工具，他里面提供了公共的代码
 */
public class Logger {

    /**
     * 用于打印日志：计划让其在切入点方法执行前执行（切入点方法就是业务层方法）
     */
    public void printLog(){
        System.out.println("Logger 类中的 printLog 方法开始记录日志了。");
    }
}

```

#### 2. 目标程序

```
public class AccountServiceImpl implements AccountService {
    public void saveAccount() {
        System.out.println("执行了保存");
    }

    public void updateAccount(int i) {
        System.out.println("执行了更新");
    }

    public int deleteAccount() {
        System.out.println("执行了删除");
        return 0;
    }
}
```

#### 3. 配置切面，确定连接点，设置切入点和通知
```
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/aop
        https://www.springframework.org/schema/aop/spring-aop.xsd">

    <!-- 配置 spring 的 IOC，把 service 对象配置进来-->
    <bean id="accountService" class="com.manman.service.impl.AccountServiceImpl"></bean>

    <!-- spring 中基于 xml 的 AOP 配置步骤
        1. 把通知 bean 也交给 spring 管理
        2. 使用 aop:config 标签标识开始 AOP 的配置
        3. 使用 aop:aspect 标签表明配置切面
            id 属性：给切面提供一个唯一标识
            ref 属性：是指定通知类的 id。
        4. 在 aop:aspect 标签的内部使用对应的标签来配置通知的类型
           我们现在示例是让 pringLog 方法在切入点方法执行之前执行，所以是前置通知。
           aop:before：表示配置前置通知。
                method 属性：用于指定 Logger 类中哪个方法是前置通知。
                pointcut 属性：用于指定切入点表达式，该表达式的含义指的是对业务层中哪些方法增强
                    关键字：execution(表达式)
                    表达式：访问修饰符 返回值 包名 类名 方法名（参数列表）
                    public void com.manman.service.impl.AccountServiceImpl.saveAccount()
    -->

    <!-- 配置 Logger 类-->
    <bean id="logger" class="com.manman.utils.Logger"></bean>
    <!-- 配置 AOP -->
    <aop:config>
        <!-- 配置切面 -->
        <aop:aspect id="logAdvice" ref="logger">
            <!-- 配置通知的类型，并且建立通知方法和切入点方法的关联 -->
            <aop:before method="printLog" pointcut="execution(public void com.manman.service.impl.AccountServiceImpl.*)"></aop:before>
        </aop:aspect>
    </aop:config>
</beans>
```
#### 4. 测试

```
public class Test {
    public static void main(String[] args) {
        // 1. 获取容器
        ApplicationContext ac = new ClassPathXmlApplicationContext("beans.xml");
        // 2. 获取对象
        AccountService as = (AccountService)ac.getBean("accountService");
        // 3. 执行方法
        as.saveAccount();
    }
}
```
执行流程 
----

在运行中调用 as.saveAccount() 方法，会先找到其对象 AccountService，判断该对象是否是某个切面对象的目标对象。若是的话，则会在内存中开辟一个空间，创建一个代理对象，该对象有目标对象的所有方法（即连接点），而被调用的方法为切入点。在代理对象的对应切入点方法中将切面和切入点进行织入。调用代理对象的对应切入点方法。


