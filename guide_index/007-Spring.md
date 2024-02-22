> # Spring容器初始化流程

参考文章：

[《源码导读-Spring容器初始化流程-XmlBeanFactory》](https://juejin.cn/post/7328012551756283915)

[《源码导读-Spring容器初始化流程-ApplicationContext》](https://juejin.cn/post/7328012551756316683)



> # Autowired和Resource关键字的区别


都是做bean注入时用的。不同的是Autowired是Spring的注解，Resource是javax的注解。Autowired是先按类型注入去查找的，找到一个直接赋值；找到多个，则再按名字去找；找不到则报错。我们可以搭配Qualifier，解决找到多个实例的问题。

Resource默认是按名字注入去查找的，他有2个参数，分别是name和type，他查到多个是会报错的。

> # SpringBean的初始化过程





> # 依赖注入的方式有几种

参考官网的说法，是有2种，分别是1、构造方法注入2、setter方法注入。在平常的开发中，有一种说法叫属性注入，也就是在属性上加入Autowired注解。



> # 谈谈你对Spring的AOP理解

aop，面向切面编程。SpringAOP是基于动态代理实现的，有两种实现方式，分别是jdk动态代理和cglib，jdk动态代理只能代理类，而cglib可以代理接口。







> # 说说你对Spring的IOC是怎么理解的

IOC控制反转。举个例子，你去洗脚，平常是你自己点小姐姐的，可是有一天，你不想自己点了，你把自己的要求告诉了老板，让老板帮你安排，相当于你把点小姐姐的权利交给了老板，这个就叫做控制反转了。而DI是依赖注入，他是控制反转的一种实现方式。







> # 解释一下spring bean的生命周期







> # Spring框架中都用到了哪些设计模式







> # Spring 是怎么解决循环依赖的

首先我们明确什么是循环依赖。其次构造器依赖Spring是没有解决的，Spring只解决了setter的循环依赖问题。



他是使用3级缓存来解决循环依赖的，说人话就是使用3个map来解决的。

第一级缓存，存放完全初始化好了的Bean，也就是大名鼎鼎的单例池；

第二级缓存，存放原始的Bean对象，这个时候Bean里的属性还没有赋值；

第三级缓存，存放Bean工厂对象，用来生产原始Bean对象，并放入二级缓存中。



我们假设现在有这样的场景AService依赖BService，BService依赖AService

1. AService首先实例化，实例化通过ObjectFactory半成品暴露在三级缓存中

2. 填充属性BService，发现BService还未进行过加载，就会先去加载BService

3. 再加载BService的过程中，实例化，也通过ObjectFactory半成品暴露在三级缓存

4. 填充属性AService的时候，这时候能够从三级缓存中拿到半成品的ObjectFactory

   

   拿到ObjectFactory对象后，调用ObjectFactory.getObject()方法最终会调用getEarlyBeanReference()方法，getEarlyBeanReference这个方法主要逻辑大概描述下如果bean被AOP切面代理则返回的是beanProxy对象，如果未被代理则返回的是原bean实例，这时我们会发现能够拿到bean实例(属性未填充)，然后从三级缓存移除，放到二级缓存earlySingletonObjects中，而此时B注入的是一个半成品的实例A对象，不过随着B初始化完成后，A会继续进行后续的初始化操作，最终B会注入的是一个完整的A实例，因为在内存中它们是同一个对象。



按道理如果只是为了解决循环依赖的话，二级缓存是完全足够了的。网上有一种流传很广的错误认知，说是Spring之所以要使用3级缓存，是为了解决AOP代理对象的问题，因为假设只有一级和三级缓存的话，我们每次从三级缓存中拿到singleFactory对象，执行getObject()方法又会产生新的代理对象，所以要用三级缓存，这种推论犯了一个逻辑错误，因为我是这么解决问题的，所以我必须要这么解决。

如果只是为了解决AOP代理对象的问题，那完全可以加个缓存判断，如果是之前织入过的，直接返回就行。所以我觉得Spring选择三级缓存来解决循环依赖，归根究底是spring设计原则的问题。

如果 Spring 选择二级缓存来解决循环依赖的话，那么就意味着所有 Bean 都需要在实例化完成之后就立马为其创建代理，而 Spring 的设计原则是在 Bean 初始化完成之后才为其创建代理。所以，Spring 选择了三级缓存。但是因为循环依赖的出现，导致了 Spring 不得不提前去创建代理，因为如果不提前创建代理对象，那么注入的就是原始对象，这样就会产生错误。

> # Bean的生命周期





> # BeanFactroy和FactroyBean的区别 

BeanFactroy是所有Spring Bean容器的顶级接口，提供了getBean这样的方法从容器中获取指定的Bean实例。BeanFactroy在生产Bean的同时，还提供了解决Bean之间的依赖注入的能力，也就是DI。 FactroyBean是一个工厂Bean，它是一个接口，主要功能是动态生成某一个类型的Bean实例，他有一个getObject方法  



> # Spring中有哪些方式可以把Bean注入到IOC容器中

1、使用xml的方式申明Bean定义

2、@ComponentScan扫描@Service等

3、使用@Configuration+@Bean

4、使用@Import导入配置类或者普通的Bean

5、使用FactroyBean动态构建Bean实例

6、实现ImportBeanDefinitionRegistrar接口的registerBeanDefinitions方法动态注册Bean，这个在SpringBoot的启动注解里有用到，mybatis扫描mapper接口也有用到。

7、使用ImportSelector接口，动态批量注入配置类或者Bean对象，这个在SrpingBoot里面的自动装配机制里面有用到。



> # 谈谈你对Spring事务的理解