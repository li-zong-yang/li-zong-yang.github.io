# SpringBoot

## ✅SpringBoot的自动装配原理

自动装配，简单来说就是自动把第三方组件的Bean装载到Spring lOC器里面，不需要开发人员再去写Bean的装配配置。

在Spring Boot应用里面，只需要在启动类加上@SpringBootApplication注解就可以实现自动装配。

@SpringBootApplication是一个复合注解，真正实现自动装配的注解是@EnableAutoConfiguration。

自动装配的实现主要依靠三个核心关键技术

引入Starter启动依赖组件的时候，这个组件里面必须要包含@Configuration配置类，在这个配置类里面通过@Bean注解声明需要装配到IOC容器的Bean对象。

这个配置类是放在第三方的jar包里面，然后通过SpringBoot中的约定优于配置思想，把这个配置类的全路径放在classpath:/META-INF/spring.factories文件中。这样SpringBoot就可以知道第三方jar包里面的配置类的位置，这个步骤主要是用到了Spring里面的SpringFactoriesLoader来完成的。

SpringBoot拿到所第三方jar包里面声明的配置类以后，再通过Spring提供的ImportSelector接口，实现对这些配置类的动态加载。

在我看来，SpringBoot是约定优于配置这一理念下的产物，所以在很多的地方，都会看到这类的思想。它的出现，让开发人员更加聚焦在了业务代码的编写上，而不需要去关心和业务无关的配置。

## ✅一个 SpringBoot 项目能处理多少请求？

一个未进行任何特殊配置，全部采用默认设置的 SpringBoot 项目，这个项目同一时刻最多能同时处理多少请求，取决于我们使用的 web 容器，而 SpringBoot 默认使用的是 Tomcat。

Tomcat 的默认核心线程数是 10，最大线程数 200，队列长度是无限长。但是由于其运行机制和 JDK 线程池不一样，在核心线程数满了之后，会直接启用最大线程数。所以，在默认的配置下，同一时刻，可以处理 200 个请求。

在实际使用过程中，应该基于服务实际情况和服务器配置等相关消息，对该参数进行评估设置。