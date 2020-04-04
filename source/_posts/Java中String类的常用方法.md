---
title: String类的常用方法
date: 2019-10-15 19:51:31
tags: Java
---
### String类的常用方法  

> **split("reg")**  
> 将字符串以reg为分隔符进行分割，返回一个String类型的数组    

> **replace(char oldChar, char newChar)**  
> 替换字符串中的字符，返回一个新的String类型的变量  

> **replaceAll(String regex,String replacement)**  
> 基于正则表达式的替换,比如，可以通过<code>replaceAll("\\d", "*")</code>把一个字符串所有的数字字符都换成星号;如果参数不是基于正则表达式的，那么效果和replace()相同

> **replaceFirst(String regex,String replacement)**  
> 基于正则表达式的替换,只替换第一次出现的

> **trim()**  
> 去除字符串首尾的空格，返回一个新的String类型的变量  

> **equals()**  
> 比较两个String类型变量的内容是否相同，区分大小写  

> **equalsIgnoreCase()**  
> 和equals()方法的作用相同，只是忽略大小写  

> **substring(fromTndex,toIndex)**  
> 字符串截取，包含包含起始位置，不包含结束位置  
> 如果只有起始位置，没有结束位置，就返回从起始位置到字符串的末尾  

> **charAt(int index)**  
> 返回字符串中指定位置的字符，Char类型   

> **boolean statWith(String prefix)**  
> 判断字符串是是否是以prefix为开始，是返回true，否返回false  

> **boolean endWith(String suffix)**  
> 判断字符串是是否是以suffix为结束，是返回true，否返回false  

> **String toLowerCase()**  
> 返回将当前字符串中所有字符转换成小写后的新串

> **String toUpperCase()**  
> 返回将当前字符串中所有字符转换成大写后的新串  

> **contains(String str)**  
> 判断参数s是否被包含在字符串中，是返回true，否返回false  

> **int indexOf(int ch/String str)**  
> 用于查找当前字符串中字符或子串，返回字符或子串在当前字符串中从左边起首次出现的位置，若没有出现则返回-1

> **indexOf(int ch/String str, int fromIndex)**  
> 从fromIndex位置向后查找  

> **lastIndexOf(int ch/String str)**  
> 从字符串的末尾位置向前查找  

### 字符串与基本类型的转换  
<pre>
int n = Integer.parseInt("12");  
float f = Float.parseFloat("12.34");  
double d = Double.parseDouble("1.124");
</pre>  

<pre>
String a = String.valueOf(12);
String b = String.valueOf(13.14);
</pre>