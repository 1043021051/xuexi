SpringBoot自动配置原理
----
以下引自知乎
```
SpringBoot 它的外表轻巧简单，但是它的里面就像一只巨大的怪兽，这只怪兽有千百只脚把自己缠绕在一起，把爱研究源码的读者绕的晕头转向。但是这 Java 编程的世界 SpringBoot 就是老大哥，你却不得不服。即使你的心中有千万头草泥马在奔跑，但是它就是天下第一。如果你是一个学院派的程序员，看到这种现象你会怀疑人生，你不得不接受一个规则 —— 受市场最欢迎的未必就是设计的最好的，里面夹杂着太多其它的非理性因素。
```
### @SpringBootApplication注解
对这个注解详细大家一定非常熟悉了。再来好好看看这个注解。  

我们点进该注解，发现它由多个注解构成。这种注解 注解注解的方式实在看着让人头疼@ComponentScan 就不多赘述了，就是一个自动扫描的注解。应该都很熟悉

我们主要看这两个SpringBoot的注解，也就是`` @SpringBootConfiguration``和``@EnableAutoConfiguration``

@SpringBootConfiguration注解发现他里面没有太多东西，只是对 @Configuration注解进行了一个封装。

简而言之，它就是一个 @Configuration 注解而已。
