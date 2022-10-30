---
title: unix时间戳转时间
categories:
  - 后端
tags: 时间戳
abbrlink: 3b7df5c1
date: 2019-11-05 19:51:31
---

```java
		String unixTimeStr=  "1572251400";
		long s = Long.parseLong(unixTimeStr);
		s *= 1000; // 乘以1000，变成毫秒
		SimpleDateFormat t = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
		String date = t.format(new Date(s));
		System.out.println(date); //2019-10-28 16:30:00
```