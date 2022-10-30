---
title: HashMap详解-源码分析
categories:
  - 后端
  - Java
tags:
  - Java
  - JDK
abbrlink: 9ca5d950
date: 2020-05-26 20:51:31
---

# HashMap详解-源码分析

## HashMap类声明

源码如下：

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable 
```

分析：

- 继承了AbstractMap抽象类，而AbstractMap类已经实现了Map部分接口，可以直接使用
- 实现了Map、Cloneable、Serializable接口，即实现Map接口中的相应方法，支持clone，支持序列化及反序列化



## 成员变量

源码如下：

```java

    private static final long serialVersionUID = 362498820763181265L;

    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

    static final int MAXIMUM_CAPACITY = 1 << 30;

    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    static final int TREEIFY_THRESHOLD = 8;

    static final int UNTREEIFY_THRESHOLD = 6;

    static final int MIN_TREEIFY_CAPACITY = 64;

 	transient Node<K,V>[] table;

 	transient Set<Map.Entry<K,V>> entrySet;

	transient int size;

 	transient int modCount;

  	int threshold;

 	final float loadFactor;

```

分析：

- serialVersionUID：序列化版本号
- DEFAULT_INITIAL_CAPACITY：默认初始化的数组容量
- MAXIMUM_CAPACITY：最大的容量，1>>30即为2的30次方
- DEFAULT_LOAD_FACTOR：默认的扩容因子
- TREEIFY_THRESHOLD：树化的阈值
- UNTREEIFY_THRESHOLD：退化为链表的阈值
- MIN_TREEIFY_CAPACITY： 树的最小容量
- table：用于保存键值对元素的数组
- entrySet：存放具体元素的集合
- size：实际存储元素的个数
- modCount：记录数组长度变化
- threshold：临界值 当实际大小(容量*扩容因子)超过临界值时，会进行扩容
- loadFactor：扩容因子



