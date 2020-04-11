---
title: Java中的代理
date: 2020-04-08 19:20:33
tags: java 代理  动态代理
---

###  Java中的代理

	   代理模式（Proxy）是通过代理对象访问目标对象，这样可以在目标对象基础上增强额外的功能，如添加权限，访问控制和审计等功能。 

- 增加额外功能，进行增强
- 引入第三方代理类，进行解耦

> 1. **静态代理**

请参考下面的代码：

```java
/**
*声明一个明星接口，明星只需要关注自己的唱、跳
*/
public interface StarService {
    void sing();
    void jump();
}
```



```java
/**
*明星接口的具体实现类
*/
public class StarServiceImpl implements StarService{
    public void sing() {
        System.out.println(" sing a song ");
    }

    public void jump() {
        System.out.println(" just dance ");
    }
}
```



```java
/**
*代理类
*/
public class StarServiceProxy implements StarService {

    private StarService starService;
	// 通过构造方法将目标实现类注入
    public StarServiceProxy(StarService starService) {
        this.starService = starService;
    }

    public void sing() {
        System.out.println("唱歌之前先联系综艺节目，确定时间、地点");
        starService.sing();
        System.out.println("表演结束之后，安排下一场演出");
    }

    public void jump() {
        return starService.jump();
        System.out.println("跳舞之后，观众鼓掌");
    }
}
```



```java

public class ProxyTest {

    public static void main(String[] args) {
        StarService starService = new StarServiceImpl();
        StarServiceProxy proxy = new StarServiceProxy(starService);
        proxy.sing();
    }
}
```



```java
输出：
 唱歌之前先联系综艺节目，确定时间、地点
  sing a song 
 表演结束之后，安排下一场演出
```




> **总结**：
>
> -  静态代理模式在不改变目标对象的前提下，实现了对目标对象的功能扩展；
> - 不足： 静态代理**实现了目标对象的所有方法**，一旦目标接口增加方法，代理对象和目标对象都要进行相应的修改，增加维护成本。 



> 2. **动态代理**

Java中的动态代理需要`java.lang.reflect.InvocationHandler`接口和` java.lang.reflect.Proxy` 类的支持 。**`jdk`的动态代理是面向接口的，即`jdk`的动态代理只能代理实现接口的类，不能代理抽象类或者没有实现接口的类。**

 `java.lang.reflect.InvocationHandler`接口的定义如下： 

```java
/**
*	Object proxy   被代理的对象  
*	Method method  要调用的方法  
*	Object[] args  方法调用时所需要参数  
*/
public interface InvocationHandler {  
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable;  
}
```

`java.lang.reflect.Proxy`类的定义如下： 

```java
/**
*	CLassLoader loader    类的加载器 
*	Class<?> interfaces   得到全部的接口  
*	InvocationHandler h   得到InvocationHandler接口的子类的实例
*/  
    public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h) throws IllegalArgumentException  

```

`jdk`动态代理流程：

1.  通过实现 `InvocationHandler `接口创建自己的调用处理器； 
2.  通过为 Proxy 类指定` ClassLoader` 对象和一组` interface `来创建动态代理类； 
3.  通过反射机制获得动态代理类的构造函数，其唯一参数类型是调用处理器接口类型； 
4.  通过构造函数创建动态代理类实例，构造时调用处理器对象作为参数被传入。 

```java
// 自定义中间代理类
public class YourHandler implements InvocationHandler {
    // 目标对象  
    private Object targetObject;  
    
    public ProxyHandler( Object proxied ) {   
    	this.proxied = proxied;   
  	}  
    
     /*proxy表示代理目标对象，method表示原对象被调用的方法，args表示方法的参数*/  
    @Ovrride
    public Object invoke( Object proxy, Method method, Object[] args ) throws Throwable {   
    //在转调具体目标对象之前，可以执行一些功能处理

    //转调具体目标对象的方法
    return method.invoke( proxy, args);  
    
    //在转调具体目标对象之后，可以执行一些功能处理
  		}    
    }
```

实际应用时代码如下：

```java
// RealSubject  要代理的目标对象
RealSubject real = new RealSubject(); 
// Subject 目标对象实现的接口
    Subject proxySubject =   	(Subject)Proxy.newProxyInstance(Subject.class.getClassLoader(), 
     new Class[]{Subject.class}, 
     new YourHandler(real));
 // proxySubject 为中间代理类        
    proxySubject.method( Object[] args);
```



> 3. **`Cglib`代理**

​        Cglib代理补充了jdk中只面向接口进行代理的不足，Cglib代理支持基于对象代理，不用关心该对象是否实现了某个接口。 Cglib是一个java字节码的生成工具，它动态生成一个被代理类的子类，子类重写被代理的类的所有不是final的方法。在子类中采用方法拦截的技术拦截所有父类方法的调用，顺势织入横切逻辑。 

以上面的`StarServiceImpl`为被代理对象，举个例子：

```java
/**
*  自定义中间代理类
*/
public class MyMethodInterceptor implements MethodInterceptor {

	@Override
	public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
		// TODO Auto-generated method stub
		System.out.println("Before: "  + method.getName());
        Object object = methodProxy.invokeSuper(o, objects);
        System.out.println("After: " + method.getName());
        return object;
	}
}
```

```java
// 测试如下
public static void main(String[] args) {
		Enhancer enhancer = new Enhancer();
        //继承被代理类
        enhancer.setSuperclass(Star.class);
        //设置回调
        enhancer.setCallback(new MyMethodInterceptor());
        //设置代理类对象
        StarServiceImpl starService = (StarServiceImpl) enhancer.create();
        //在调用代理类中方法时会被我们实现的方法拦截器进行拦截
        starService.sing();
	}
```

```java
输出结果如下：
    Before: sing
	sing a song
	After: sing
```

