---
title: Vue项目中实现回车登录
categories:
  - 前端
tags:
  - JavaScript
  - Vue
abbrlink: 8184ba83
date: 2020-04-04 18:51:31
---
### Vue项目中实现回车登录

```javaScript
created(){
	let that = this;
    document.onkeypress = function(e) {
      var keycode = document.all ? event.keyCode : e.which;
      if (keycode == 13) {// 回车键对应值为13
        that.login();// 登录方法名
        return false;
      }
    };
   }
```

