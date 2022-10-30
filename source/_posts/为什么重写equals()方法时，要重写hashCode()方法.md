---
title: 为什么重写equals()方法时，要重写hashCode()方法
categories:
  - 后端
  - Java
tags: Java
abbrlink: 300023f4
date: 2020-04-07 20:51:31
---

## 为什么重写equals()方法时，要重写hashCode()方法

### 1. `Object`类中`equals()`方法源代码如下所示：

```java
/**
*	Object类中的equals()方法
*/
public boolean equals(Object obj) {
        return (this == obj);
    }
```

>  由以上源代码知，`Object`类中的`equals()`方法是直接使用`==`运算符来判断两个对象相等的。
>
> - 引用类型变量使用`==`时，比较的是引用类型变量指向的对象的内存地址
> - 基本类型使用`==`时，比较值



`Objcect`类中的`hashCode`源代码如下：

```java
 /**
     * Returns a hash code value for the object. This method is
     * supported for the benefit of hash tables such as those provided by
     * {@link java.util.HashMap}.
     * <p>
     * The general contract of {@code hashCode} is:
     * <ul>
     * <li>Whenever it is invoked on the same object more than once during
     *     an execution of a Java application, the {@code hashCode} method
     *     must consistently return the same integer, provided no information
     *     used in {@code equals} comparisons on the object is modified.
     *     This integer need not remain consistent from one execution of an
     *     application to another execution of the same application.
     * <li>If two objects are equal according to the {@code equals(Object)}
     *     method, then calling the {@code hashCode} method on each of
     *     the two objects must produce the same integer result.
     * <li>It is <em>not</em> required that if two objects are unequal
     *     according to the {@link java.lang.Object#equals(java.lang.Object)}
     *     method, then calling the {@code hashCode} method on each of the
     *     two objects must produce distinct integer results.  However, the
     *     programmer should be aware that producing distinct integer results
     *     for unequal objects may improve the performance of hash tables.
     * </ul>
     * <p>
     * As much as is reasonably practical, the hashCode method defined by
     * class {@code Object} does return distinct integers for distinct
     * objects. (This is typically implemented by converting the internal
     * address of the object into an integer, but this implementation
     * technique is not required by the
     * Java&trade; programming language.)
     *
     * @return  a hash code value for this object.
     * @see     java.lang.Object#equals(java.lang.Object)
     * @see     java.lang.System#identityHashCode
     */
	public native int hashCode();// java8中的hashCode方法，
```

> 上面的注释中有说明如下几点：
>
> - 对象的`hashCode`值通常是根据对象的内存地址计算得来
> - 两个对象`equals()`结果为`true`时，两个对象的`hashCode`值一定相等，不同对象的`hashCode`不等
> - `native`标识此方法不是`java`语言实现



`Object`类中的`toString()`方法源代码如下：

```java
public String toString() {
    	// 从这里就能看出打印对象时不重写toString()方法时，就会打印出对象的hashCode值
        return getClass().getName() + "@" + Integer.toHexString(hashCode());
    }
```

### 2. `String`类中`equals()`方法和`hashCode()`方法

> `String`类中部分源代码如下所示：

```java
  
  /** The value is used for character storage. */
    private final char value[];
  /** Cache the hash code for the string */
    private int hash; // Default to 0
  /**
  * 无参构造方法
  */
  public String() {
        this.value = "".value;
    }
  /**
  * 有参构造方法
  */
  public String(String original) {
        this.value = original.value;
        this.hash = original.hash;
    }
  /**
  *String类重写的equals方法
  */
  public boolean equals(Object anObject) {
        if (this == anObject) {// 此处的this指向a.equals(b)的a对象，即谁调用指向谁
            return true;
        }
        if (anObject instanceof String) {
            String anotherString = (String)anObject;
            int n = value.length;
            if (n == anotherString.value.length) {
                char v1[] = value;
                char v2[] = anotherString.value;
                int i = 0;
                while (n-- != 0) {
                    if (v1[i] != v2[i])
                        return false;
                    i++;
                }
                return true;
            }
        }
        return false;
    }
   /**
     * Returns a hash code for this string. The hash code for a
     * {@code String} object is computed as
     * <blockquote><pre>
     * s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]
     * </pre></blockquote>
     * using {@code int} arithmetic, where {@code s[i]} is the
     * <i>i</i>th character of the string, {@code n} is the length of
     * the string, and {@code ^} indicates exponentiation.
     * (The hash value of the empty string is zero.)
     *
     * @return  a hash code value for this object.
     */
    public int hashCode() {
        int h = hash;
        if (h == 0 && value.length > 0) {
            char val[] = value;

            for (int i = 0; i < value.length; i++) {
                h = 31 * h + val[i];
            }
            hash = h;
        }
        return h;
    }

```

从上面的源码中，我们不难发现`String`类已经重写了`equals()`方法和`hashCode()`方法。

`String`类重写的`equals()`方法判断流程如下：

> 1. 使用`==`来判断两个对象的内存地址是否相同，相同返回`true；`
> 2. 如果两个对象的内存地址不同，程序继续往下走，判断另一个对象是否是`String`类型的；
> 3. 如果比较对象不是`String`类型，直接返回`false`；
> 4. 如果是`String`类型的，进行类型强转；
> 5. 比较两个`String`的字符数组长度，如果长度不同，返回`false`；
> 6. 利用`while`循环来逐位比较字符是否相等，直到循环结束，所有字符都相等，则返回`true`，否则返回`false`;

下面来看一下重写的`hashCode()`方法。

> 1. 首先`String`类中定义了一个`int`类型的变量`hash`用来缓存`String`对象的`hash`值；
> 2. 如果当前调用`hashCode()`方法的`String`对象在常量池没有找到，并且该对象的`length`长度大于`0`,则继续往下走，否则返回`0`;即`String`类默认`""`字符串的`hashCode()`值为`0`;
> 3. 遍历字符数组，获取每一个字符的`ASCII`码表对应的值 和之前的`hash`值相加，这样就保证了相同的字符串的`hashCode()`返回值相同，计算公式在注释里已经写出来了：**`s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]`**
> 4. 将计算出来的结果保存到`hash`变量中，并返回该值；

这里为什么要乘以`31`呢？原因是为了**性能**，不仅仅指降低了计算速度，也降低了哈希冲突的概率。

>  哈希冲突：此处指不同的字符串生成了相同的`hashCode`值。

`31`是一个**奇素数**。如果乘数是偶数，并且乘法溢出的话，信息就会丢失，因为与2相乘等价于移位运算（低位补`0`）。使用素数的好处并不很明显，但是习惯上使用素数来计算散列结果。 **`31` 有个很好的性能，即用移位和减法来代替乘法**，可以得到更好的性能： `31 * i == (i << 5）- i`， 现代的 `VM`  可以自动完成这种优化。这个公式可以很简单的推导出来。 ---- 《`Effective Java`》 

> 素数：质数又称素数，指在一个大于1的自然数中，除了1和此整数自身外，没法被其他自然数整除的数。 

### 说了这么多，为什么重写equals()方法要重写hashCode()方法呢？

原因如下：

- <font color="red">**避免两个对象不同时出现相同的hashCode值**</font>
- <font color="red">**避免重写过equals()方法的对象使用equals()判断为相等时，返回不同的hashCode值**</font>

