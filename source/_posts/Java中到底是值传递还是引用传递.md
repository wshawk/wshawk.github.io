---
title: Java中到底是值传递还是引用传递
date: 2019-07-15 19:51:31
tags: Java
---
## Java中到底是值传递还是引用传递？  

我们先回顾一下基本概念
### 实参和形参  

参数在编程语言中是执行程序需要的数据，这个数据一般保存在变量中。在Java中定义一个方法时，可以定义一些参数，  
举个例子：    

```java
public class Example {

public static void main(String[] args) {
	String myName = "hawk";
	sayYourName(myName);// 实际参数是myName
	}
public static void sayYourName(String name) {// 形式参数是name
		System.out.println(name);
	}  
}
```

上面的代码中定义一个名为sayYourName的方法，如果想要执行这个方法，那么你需要传入一个String类型的变量给这个方法，定义这个方法时声明的String类型的name就是形式参数，而在这个方法执行时传入的myName就是实际参数。

#### 小结  
> 实际参数是调用有参方法的时候真正传递的内容，而形式参数是用于接收实参内容的参数。  

### 传参的几种方式  
 按值传递(call by value)  
>按值传递，就是指在调用函数时，将实参对应的值做一个拷贝指向函数对应的形参。在函数内改变形参对应的值并不会影响外部实参的值  


 按引用传递 (call by reference)
>按引用传递，是指在调用函数时，传递给函数的是实参的地址即引用，而不是实参的拷贝。在函数内部参数的值，对外部的实参是可见的。  

按共享传递 （call by sharing）  
>按共享传递，是指在调用函数时，传递给函数的是实参的地址的拷贝（如果实参在栈中，则直接拷贝该值）。在函数内部对参数进行操作时，需要先拷贝的地址寻找到具体的值，再进行操作。如果该值在栈中，那么因为是直接拷贝的值，所以函数内部对参数进行操作不会对外部变量产生影响。如果原来拷贝的是原值在堆中的地址，那么需要先根据该地址找到堆中对应的位置，再进行操作。因为传递的是地址的拷贝所以函数内对值的操作对外部变量是可见的。  

>按共享传递可以理解为**按值传递**的一个特例，这里的值是对象的引用地址，而不是具体对象。  

上面描述的传参方式实际上是函数调用时参数的求值策略(Evaluation Strategy)，这个其实比较好理解，比如我们调用上面的sayYourName()方法  
我们可以这样：sayYourName("hawk");  
可以这样:sayYourName("ha"+"wk");  
可以这样：sayYourName("hawk"+x.subString(2));// 此处的x是一个变量,进行截取后和"hawk"合并    

但是所有这些实参的形式，都统称为表达式(Expression)。求值（Evaluation）即是指对这些表达式的简化并求解其值的过程。

求值策略(值传递和引用传递)的关注的点在于，这些表达式在调用函数的过程中，求值的时机、值的形式的选取等问题。求值的时机，可以是在函数调用前，也可以是在函数调用后，由被调用者自己求值。这里所谓调用后求值，可以理解为Lazy Load或On Demand的一种求值方式。
而且，除了按值传递和按引用传递，还有一些其它的求值策略。这些求值策略的划分依据是：求值的时机（调用前还是调用中）和值本身的传递方式。详见下表：  
<table>
<tr>
	<td>求值策略</td>
	<td>求值时间</td>
	<td>传值方式</td>
</tr>
<tr>
	<td>按值传递( pass by value )</td>
	<td>调用前</td>
	<td>值的结果(只是结果的副本)</td>
</tr>
<tr>
	<td>按引用传递( pass by reference )</td>
	<td>调用前</td>
	<td>原值(原始对象，无副本)</td>
</tr>
<tr>
	<td>按名传递( pass by name )</td>
	<td>调用后(用到才求值)</td>
	<td>与值无关的一个名</td>
</tr>
</table>  

**按值传递和按引用传递在行为表现上的差异如下：**  
<table>
<tr>
	<td></td>
	<td>按值传递</td>
	<td>按引用传递</td>
</tr>
<tr>
	<td>本质区别</td>
	<td>创建副本</td>
	<td>不创建副本</td>
</tr>
<tr>
	<td>表现结果</td>
	<td>函数中无法改变原始对象</td>
	<td>函数中可以改变原始对象</td>
</tr>
</table>
**这里面的改变，是指把一个变量指向另一个对象，而不是指仅仅改变属性或者成员什么的**  

只要是按值传递，不管传递的参数类型是值类型还是引用类型，都会在调用栈上面创建一个实参的副本，区别只是如果传递的参数类型是值类型，该副本就是实际参数的值的复制，而对于引用类型来说，引用类型的实例(即对象)是保存在堆中的，在栈上只有一个该实例的引用（一般情况下是该实例在堆中的内存地址），此时，实际参数的副本并不是该对象，而是引用的副本。

**所以，综上所述，对于Java的函数调用方式最准确的描述是：参数藉由值传递方式，传递的值是个引用。（句中两个“值”不是一个意思，第一个值是evaluation result，第二个值是value content）**

## 举例     
说到Java是值传递还是引用传递，一般都会举下面三个例子，我们来一一说明一下：  
第一个例子，基本类型参数传值：  

```java
public class Example {
	public static void main(String[] args) {
		int i = 10;
		changeValue(i);
		System.out.println(i);
	}
	public static void changeValue(int a ) {
		a = 20;
	}
}  
```

上面的代码解释如下：  
定义了一个int类型的变量i，并赋值为10  
调用changeValue方法，传入i，changeValue方法将传入的变量赋值为20  
输出i  

结果是：**10**    
changValue方法内部对**形参a**的修改并没有影响到**实参i**  
**第一个结论：Java中基本类型的传值方式是按值传递**  

第二个例子，引用类型传值：    

```java
public class Example {
	public static void main(String[] args) {
		String i = "hawk";
		changString(i);
		System.out.println(i);
	}
	public static void changString(String a ) {
		a = "HAWK";
	}
}    
```

上面的代码解释如下：  
定义一个String类型的变量i，赋值为"hawk"  
调用changeString方法，将传入的变量赋值为"HAWK"  
输出i  

结果： **hawk**  
在Java中String类型虽然可以直接赋值，但是是引用类型，**因为String类型的具体值保存在堆中，而不是栈上**  
既然String是引用类型， 上面的例子又说明String是值传递的  那么我们是不是就可以得出：  
**Java中引用类型也是值传递的**，这样的结论呢？  

我们来看第三个例子：  

```java
public class Example {
	int i = 20;
	public static void main(String[] args) {
		Example ex = new Example();
		changValue(ex);
		System.out.println(ex.i);
	}
	public static void changValue(Example e ) {
		e.i = 30;
	}
}  
```
上面的代码解释如下：  
实例化一个Example对象ex，ex中只有一个属性i，初始值为20  
调用changValue方法，将ex中的属性i的值赋值为30  
输出ex中属性i的值  

结果是：**30**  
这样的结果就让很多人产生了困惑，String类型是引用类型，对象也是引用类型，为什么第二个例子不能改变i的值，第三个例子却改变了ex中i的值呢？**Java中引用类型的参数传递到底是引用传递还是值传递呢？**  

其实，第二个例子和第三个例子我们所关注的点已经是错误的了，自然无法得出正确的结论   
我们回顾一下**引用传递和值传递的本质区别和行为表现上的区别** 

<table>
<tr>
	<td></td>
	<td>按值传递</td>
	<td>按引用传递</td>
</tr>
<tr>
	<td>本质区别</td>
	<td>创建副本</td>
	<td>不创建副本</td>
</tr>
<tr>
	<td>表现结果</td>
	<td>函数中<b>无法改变原始对象</b></td>
	<td>函数中<b>可以改变原始对象</b></td>
</tr>
</table>
注意到了吗？我们应该关注的是**原始对象的变化**  
> **Java中，引用类型的原始对象（即原始值）是保存在堆中的，变量中存储的是原始值的引用（Java中存储的是该对象在堆中的内存地址），**
> **该引用指向堆中的对象,所以我们<font color="red">关注的关键点应该是对象的引用有没有发生变化</font>，而不是原始对象中的内容有没有发生变化**

**<font color="red">重要依据：</font>如果可以改变对象的引用，就说明是引用传递，如果不能改变对象的引用就说明是值传递**  
下面我们多举一个例子： 

```java
public class Example {
	int i = 20;
	public static void main(String[] args) {
		Example ex = new Example();
		changValue(ex);
		System.out.println(ex.i);
	}
	public static void changValue(Example e ) {
		e = new Example();
		e.i = 50;
	}
}  
```
这个例子和上面第三个例子很相似，区别在于：  
changValue方法中将一个新的原始对象的引用赋值给了形参e  
**如果Java是按引用传递的话，e = new Example();就是修改了实参ex的引用，即就是改变了原始对象  
如果Java是按值传递的话，实参ex不会有任何变化**  



结果：**20**  

这个结果说明：**changValue方法并没有修改到ex的引用，也就是说，e只是ex的副本，对e的引用进行的所有操作都不会影响到ex，所以我们
从上述代码运行的结果也可以证明Java中是只有值传递的**
## 参考  
<a href="http://www.hollischuang.com/archives/2275" target="_blank">为什么说Java中只有值传递</a>  
<a href="http://menzhongxin.com/2017/02/07/%E6%8C%89%E5%80%BC%E4%BC%A0%E9%80%92-%E6%8C%89%E5%BC%95%E7%94%A8%E4%BC%A0%E9%80%92%E5%92%8C%E6%8C%89%E5%85%B1%E4%BA%AB%E4%BC%A0%E9%80%92/" target="_blank">按值传递、按引用传递、按共享传递</a>  
<a href="https://www.zhihu.com/question/20628016/answer/28970414" target="_blank">为什么 Java 只有值传递，但 C# 既有值传递，又有引用传递，这种语言设计有哪些好处？</a>
