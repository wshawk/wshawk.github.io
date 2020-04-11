---
title: JS中this的指向
date: 2019-10-15 19:51:31
tags: JavaScript
---
> JS中this的指向

	 1. 以函数形式调用时，this是window
	 2. 以方法形式调用时，this是调用方法的对象
	 3. 以构造函数形式调用时，this是新建的那个对象
	 4. 使用call()和apply()调用时，this是指定的那个对象
在调用函数时，浏览器每次都会传递两个隐含的参数
	1. 函数的上下文对象this	
	2. 封装实参的对象arguments
		-- arguments是一个类数组对象(不是数组)，它可以用过索引来操作数据，也可以获取长度
		-- 在调用函数时，我们所传递的实参都会保存在arguments中
		-- 即使在函数中不定义形参，也可以通过arguments来进行操作，只是相对而言麻烦一些
