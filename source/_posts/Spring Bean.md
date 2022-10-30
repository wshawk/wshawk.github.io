---
title: Spring Bean
categories:
  - 后端
tags:
  - Spring
abbrlink: cb377708
date: 2020-09-05 23:30:31
---



# Spring Bean

在Spring中，bean 是一个被实例化，组装，并通过 Spring IoC 容器所管理的**对象**。（**<font color="red">也就是由spring来进行管理的资源对象</font>**）



## 1. 如何将bean放入spring容器中

**既然要让spring来管理这些资源类，那么就需要将资源放入到容器中，这样spring容器才能将资源提供给有需要的类进行使用，那么如何将bean对象放入到spring容器中呢？**



有以下三种方法：

- **<font color="red">Java config配置</font>**

使用Java config的方式进行配置时，不需要创建额外的xml文件，需要创建一个java的config类。

```java
package byJavaConfig;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration // 表明是一个配置类
@ComponentScan(basePackages = "byJavaConfig") // 扫描路径
public class SpringBeanConfig {

    @Bean(name = "helloworld")
    public HelloWorld getHelloWorld() {
        return new HelloWorld();
    }
}
```

bean对象的代码：

```java
  
package byJavaConfig;

public class HelloWorld {
    private String message;

    public void setMessage(String message) {
        this.message = message;
    }

    public void getMessage() {
        System.out.println("Hello Spring By JavaConfig");
    }
}
```



测试运行代码：

```java
package byJavaConfig;

import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class Run {

    public static void main(String[] args) {
        // 通过IOC容器获得Bean
        ApplicationContext context =
                new AnnotationConfigApplicationContext(SpringBeanConfig.class);
        HelloWorld helloWorld = (HelloWorld)context.getBean("helloworld");
        helloWorld.getMessage();
    }
}
```



- **<font color="red">xml配置</font>**

使用xml配置，需要创建一个xml文件，命名可以进行自定义，示例代码如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="helloWorld" class="byXML.HelloWorld">
    </bean>
</beans>
```

XML 格式配置文件的根元素是 `<beans>`，该元素包含了多个` <bean> `子元素，每一个` <bean>` 子元素定义了一个 Bean，并描述了该 Bean 如何被装配到 Spring 容器中。

bean对象的代码：

```java
package byXML;

public class HelloWorld {
    private String message;

    public void setMessage(String message) {
        this.message = message;
    }

    public void getMessage() {
        System.out.println("Hello Spring By XML");
    }
    
}
```

测试运行代码：

```java
package byXML;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
public class Run {
    public static void main(String[] args) {
        ApplicationContext context =
                new ClassPathXmlApplicationContext("spring-XML.xml");
        HelloWorld obj = (HelloWorld) context.getBean("helloWorld");
        obj.getMessage();
    }
}
```



- **<font color="red">注解配置</font>**

注解配置也是需要一份xml的配置文件的，如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
         http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
         http://www.springframework.org/schema/context
         http://www.springframework.org/schema/context/spring-context-3.0.xsd"
>
    <context:component-scan base-package="byZHUJIE"/>
	<!-- 声明包扫描路径 -->
</beans>
```

Bean对象：

```java
package byZHUJIE;

import org.springframework.stereotype.Component;

@Component("hello")
public class HelloWorld {
    private String message;

    public void setMessage(String message) {
        this.message = message;
    }

    public void getMessage() {
        System.out.println("Hello Spring By 注解");
    }
}
```

测试类：

```java
package byZHUJIE;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class Run {
    public static void main(String[] args) {
        ApplicationContext ctx = new ClassPathXmlApplicationContext
                ("spring-ZHUJIE.xml");
        HelloWorld helloWorld = (HelloWorld) ctx.getBean("hello");
        helloWorld.getMessage();
    }
}
```



## 2. Bean的作用域

Spring 容器在初始化一个 Bean 的实例时，同时会指定该实例的作用域。Spring3 为 Bean 定义了五种作用域，具体如下。

#### 1）singleton

单例模式，使用 singleton 定义的 Bean 在 Spring 容器中只有一个实例，这也是 Bean 默认的作用域。

#### 2）prototype

原型模式，每次通过 Spring 容器获取 prototype 定义的 Bean 时，容器都将创建一个新的 Bean 实例。

#### 3）request

在一次 HTTP 请求中，容器会返回该 Bean 的同一个实例。而对不同的 HTTP 请求，会返回不同的实例，该作用域仅在当前 HTTP Request 内有效。

#### 4）session

在一次 HTTP Session 中，容器会返回该 Bean 的同一个实例。而对不同的 HTTP 请求，会返回不同的实例，该作用域仅在当前 HTTP Session 内有效。

#### 5）global Session

在一个全局的 HTTP Session 中，容器会返回该 Bean 的同一个实例。该作用域仅在使用 portlet context 时有效。



## 3. Bean的生命周期

<font color="#FF3030" size="3">注意：Spring 只帮我们管理单例模式 Bean 的完整生命周期,对于 prototype 的 bean ,Spring 在创建好交给使用者之后则不会再管理后续的生命周期。</font>

1、实例化一个Bean－－也就是我们常说的new；

2、按照Spring上下文对实例化的Bean进行配置－－也就是依赖注入；

3、如果这个Bean已经实现了BeanNameAware接口，会调用它实现的setBeanName(String)方法，此处传递的就是Spring配置文件中Bean的id值

4、如果这个Bean已经实现了BeanFactoryAware接口，会调用它实现的setBeanFactory(setBeanFactory(BeanFactory)传递的是Spring工厂自身（可以用这个方式来获取其它Bean，只需在Spring配置文件中配置一个普通的Bean就可以）；

5、如果这个Bean已经实现了ApplicationContextAware接口，会调用setApplicationContext(ApplicationContext)方法，传入Spring上下文（同样这个方式也可以实现步骤4的内容，但比4更好，因为ApplicationContext是BeanFactory的子接口，有更多的实现方法）；
<font color="#FF3030" size="3">注：去掉此步骤就是BeanFactory的Bean的生命周期</font>

6、如果这个Bean关联了BeanPostProcessor接口，将会调用postProcessBeforeInitialization(Object obj, String s)方法，BeanPostProcessor经常被用作是Bean内容的更改，并且由于这个是在Bean初始化结束时调用那个的方法，也可以被应用于内存或缓存技术；

7、如果Bean在Spring配置文件中配置了init-method属性会自动调用其配置的初始化方法。

8、如果这个Bean关联了BeanPostProcessor接口，将会调用postProcessAfterInitialization(Object obj, String s)方法、；

<font color="#FF3030" size="3">注：以上工作完成以后就可以应用这个Bean了，那这个Bean是一个Singleton的，所以一般情况下我们调用同一个id的Bean会是在内容地址相同的实例</font>

9、当Bean不再需要时，会经过清理阶段，如果Bean实现了DisposableBean这个接口，会调用那个其实现的destroy()方法；

10、最后，如果这个Bean的Spring配置中配置了destroy-method属性，会自动调用其配置的销毁方法



简单来说就四步

- 实例化
- 属性赋值
- 初始化
- 销毁

其余的操作都是在穿插在这四步之中