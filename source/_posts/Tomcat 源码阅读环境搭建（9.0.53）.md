---
title: Tomcat 源码阅读环境搭建（9.0.53）
categories:
  - 后端
tags:
  - Tomcat
abbrlink: b5f2cfa1
date: 2022-04-12 23:30:31
---


# Tomcat 源码阅读环境搭建（9.0.53）



## 1. 下载源码包

打开[tomcat官网](https://tomcat.apache.org/)

![](https://z3.ax1x.com/2021/11/17/I53pgx.png#id=jwJv2&originHeight=780&originWidth=1184&originalType=binary&ratio=1&status=done&style=none)

![I51v59[1].png](https://cdn.nlark.com/yuque/0/2021/png/12460206/1637144111523-dbce0bf9-d82e-4fd4-83e3-721f3b846b92.png#clientId=u82453ed0-17bb-4&from=ui&id=u3fa4eb7c&originHeight=460&originWidth=1170&originalType=binary&ratio=1&size=20050&status=done&style=none&taskId=u181800dc-c160-41db-a952-959fdd7147f)
![](https://z3.ax1x.com/2021/11/17/I51zCR.png#id=ISHF3&originHeight=958&originWidth=1254&originalType=binary&ratio=1&status=done&style=none)
![image-20211108113918132.png](https://cdn.nlark.com/yuque/0/2021/png/12460206/1637141553397-a9edda7f-13e4-439f-8836-9ffff3e4dde6.png#clientId=uf44feb28-a9d2-4&from=paste&height=920&id=u35b00f0f&originHeight=920&originWidth=1517&originalType=binary&ratio=1&size=108349&status=done&style=none&taskId=u6ddb6584-8914-4f38-a8eb-0d693b57b21&width=1517)
![image-20211108113947876.png](https://cdn.nlark.com/yuque/0/2021/png/12460206/1637141566639-3b26639d-2b2a-4518-8c28-6bc229617e13.png#clientId=uf44feb28-a9d2-4&from=paste&height=471&id=u438dc29d&originHeight=471&originWidth=1307&originalType=binary&ratio=1&size=25325&status=done&style=none&taskId=ude21f5a2-a34a-4c38-a89b-953bfadede4&width=1307)
## 2. 解压之后，用 idea 打开文件夹

![image-20211108114229791.png](https://cdn.nlark.com/yuque/0/2021/png/12460206/1637141584945-8c954726-eacc-4f73-a908-2b8dc10f8492.png#clientId=uf44feb28-a9d2-4&from=paste&height=805&id=u464beb23&originHeight=805&originWidth=1707&originalType=binary&ratio=1&size=286877&status=done&style=none&taskId=u55e12550-6466-495f-a507-11a7f2b5104&width=1707)

**tomcat是使用ant来进行管理的，为方便使用，引入maven来处理依赖**

![image-20211108114423745.png](https://cdn.nlark.com/yuque/0/2021/png/12460206/1637141595053-ffc68703-b27d-4b1e-a7ad-ec5de34e76e9.png#clientId=uf44feb28-a9d2-4&from=paste&height=921&id=u79d86534&originHeight=921&originWidth=1534&originalType=binary&ratio=1&size=229047&status=done&style=none&taskId=u7c91cd1d-de4a-4281-beba-a2ebc03b76e&width=1534)

![image-20211108114847968.png](https://cdn.nlark.com/yuque/0/2021/png/12460206/1637141606149-eee8d426-b2fe-407f-bb8b-1a06ceef91ba.png#clientId=uf44feb28-a9d2-4&from=paste&height=636&id=u847b3aa7&originHeight=636&originWidth=748&originalType=binary&ratio=1&size=43268&status=done&style=none&taskId=u4f27782c-9413-468b-8cc7-275614e53a5&width=748)

在生成的`pom.xml`文件中，粘贴以下代码

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>groupId</groupId>
    <artifactId>apache-tomcat-9.0.53-src</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>
    <dependencies>
      <dependency>
        <groupId>org.apache.tomcat</groupId>
        <artifactId>catalina</artifactId>
        <version>6.0.53</version>
      </dependency>
      <dependency>
        <groupId>org.apache.ant</groupId>
        <artifactId>ant</artifactId>
        <version>1.10.11</version>
      </dependency>
      <dependency>
        <groupId>biz.aQute.bnd</groupId>
        <artifactId>biz.aQute.bnd.annotation</artifactId>
        <version>6.0.0</version>
      </dependency>
      <dependency>
        <groupId>org.eclipse.jdt</groupId>
        <artifactId>org.eclipse.jdt.core</artifactId>
        <version>3.26.0</version>
      </dependency>
      <dependency>
        <groupId>javax</groupId>
        <artifactId>javaee-api</artifactId>
        <version>8.0.1</version>
      </dependency>
      <dependency>
        <groupId>wsdl4j</groupId>
        <artifactId>wsdl4j</artifactId>
        <version>1.6.3</version>
      </dependency>
    </dependencies>
</project>
```

`java`文件上右键， 设置文件夹类型

![image-20211108140710894.png](https://cdn.nlark.com/yuque/0/2021/png/12460206/1637141619810-d4d3818c-c305-4e0b-ab2e-193a5a5c7a56.png#clientId=uf44feb28-a9d2-4&from=paste&height=865&id=u6b9beb91&originHeight=865&originWidth=1072&originalType=binary&ratio=1&size=577227&status=done&style=none&taskId=u18765b9e-338a-4e78-a03f-f96c5f40036&width=1072)

## 3. 启动tomcat

### 3.1 配置项目的JDK

![image-20211108141027786.png](https://cdn.nlark.com/yuque/0/2021/png/12460206/1637141634966-019014f7-88ac-43ad-951b-01d6ff306f64.png#clientId=uf44feb28-a9d2-4&from=paste&height=878&id=udc2461c7&originHeight=878&originWidth=1411&originalType=binary&ratio=1&size=834788&status=done&style=none&taskId=ua8c984c4-a71c-4c59-ab00-dcda706e58f&width=1411)

### 3.2 启动Bootstrap

`tomcat`的启动类是 `Bootstrap.java`， 找到这个类，点击运行

![image-20211108141254070.png](https://cdn.nlark.com/yuque/0/2021/png/12460206/1637141645637-87ce44b7-2d26-43e2-962f-cd81f2fb7064.png#clientId=uf44feb28-a9d2-4&from=paste&height=603&id=u8cd6e22d&originHeight=603&originWidth=1507&originalType=binary&ratio=1&size=704072&status=done&style=none&taskId=ubb8c83da-3e40-4853-a0b0-9508c5bcdde&width=1507)

#### 解决乱码问题

找到`org.apache.tomcat.util.res.StringManager`类

![image-20211108141357561.png](https://cdn.nlark.com/yuque/0/2021/png/12460206/1637141671105-74db153d-7902-4d4f-92f3-2a45c0c74827.png#clientId=uf44feb28-a9d2-4&from=paste&height=858&id=uf1c6c606&originHeight=858&originWidth=1287&originalType=binary&ratio=1&size=423954&status=done&style=none&taskId=u0adc8865-9236-4146-bfbe-e769ed9ada0&width=1287)

在`getString(String key)`方法中，`return str;` 之前，即整个文件的第`149`行，添加以下代码：

```java
        try{
            str = new String(str.getBytes(StandardCharsets.ISO_8859_1), StandardCharsets.UTF_8);
        }catch (Exception e){
            e.printStackTrace();
        }
```

还有`org.apache.jasper.compiler.Localizer`类，`getMessage(String errCode)`方法替换为以下代码。

```java
    public static String getMessage(String errCode) {
        String errMsg = errCode;
        try {
            if (bundle != null) {
                errMsg = bundle.getString(errCode);
            }
        } catch (MissingResourceException e) {
        }
        try{
            errMsg = new String(errMsg.getBytes(StandardCharsets.ISO_8859_1), StandardCharsets.UTF_8);
        }catch (Exception e){
            e.printStackTrace();
        }
        return errMsg;
    }
```

再次启动，乱码问题已解决。

![image-20211108141831778.png](https://cdn.nlark.com/yuque/0/2021/png/12460206/1637141681316-f8c1cf41-7dff-4628-b0c1-b6c1e7484148.png#clientId=uf44feb28-a9d2-4&from=paste&height=444&id=u867cb1ef&originHeight=444&originWidth=1648&originalType=binary&ratio=1&size=488618&status=done&style=none&taskId=u27a4a719-8905-475a-9725-b996160da88&width=1648)

#### 解决启动过程中控制台的报错

刚刚启动时，发现控制台有报错，截图如下。

![image-20211108142023232.png](https://cdn.nlark.com/yuque/0/2021/png/12460206/1637141693787-1c10f73d-95a0-46a6-86df-48b58c07742c.png#clientId=uf44feb28-a9d2-4&from=paste&height=718&id=ue0942b28&originHeight=718&originWidth=1636&originalType=binary&ratio=1&size=936600&status=done&style=none&taskId=u0f9a2736-a223-42d4-a3b6-fd488504cb6&width=1636)

百度后，了解到是`webapps`下的`examples`文件夹的问题，删掉即可。

![image-20211108142213216.png](https://cdn.nlark.com/yuque/0/2021/png/12460206/1637141694311-fa67860d-cee7-4a17-9559-e775f4593f8d.png#clientId=uf44feb28-a9d2-4&from=paste&height=431&id=ub4b9c3fa&originHeight=431&originWidth=483&originalType=binary&ratio=1&size=129640&status=done&style=none&taskId=ucedaa715-1213-4c7d-81e1-eda0183e3d4&width=483)

删掉后再次启动，控制台就没有报错信息了。

![image-20211108142256229.png](https://cdn.nlark.com/yuque/0/2021/png/12460206/1637141701462-a630ca75-b1c1-42ea-904c-942b1dc1bcf1.png#clientId=uf44feb28-a9d2-4&from=paste&height=493&id=ub505baf3&originHeight=493&originWidth=1632&originalType=binary&ratio=1&size=556458&status=done&style=none&taskId=u3d726bc7-9220-410c-b745-aa275eb7c56&width=1632)

#### 解决访问默认页面的报错

访问默认的页面`localhost:8080`时，页面显示如下：

![image-20211108153136299.png](https://cdn.nlark.com/yuque/0/2021/png/12460206/1637141756609-b3b1f605-f239-484a-bffa-89a2d46bd2ca.png#clientId=uf44feb28-a9d2-4&from=paste&height=516&id=udef10658&originHeight=516&originWidth=1253&originalType=binary&ratio=1&size=35717&status=done&style=none&taskId=u8b620c9d-6e2f-462f-8956-006ffa0551e&width=1253)

同时，控制台打印如下：

![image-20211108150034814.png](https://cdn.nlark.com/yuque/0/2021/png/12460206/1637141768058-e68dd321-2789-4fdd-8c46-48dce7ce22ba.png#clientId=uf44feb28-a9d2-4&from=paste&height=379&id=u02ab4784&originHeight=379&originWidth=1624&originalType=binary&ratio=1&size=420143&status=done&style=none&taskId=ue9f58860-6f84-4857-9f43-ee61674616d&width=1624)

`debug`到`Validator.java`的 527 行。

![image-20211108152331457.png](https://cdn.nlark.com/yuque/0/2021/png/12460206/1637141775463-30126fcc-0955-4cc6-b3c6-7f4c8d2af2c4.png#clientId=uf44feb28-a9d2-4&from=paste&height=831&id=u7040784a&originHeight=831&originWidth=1307&originalType=binary&ratio=1&size=655597&status=done&style=none&taskId=u4b6bc280-dfd7-4317-90a2-1f16ba6c396&width=1307)

这里获取到的`DefaultFactory`是 `null`， 找一下`DefaultFactory`是什么时候放进去的，全局搜索

![image-20211108152535561.png](https://cdn.nlark.com/yuque/0/2021/png/12460206/1637141781996-bb02c5d9-95da-4f5f-b4ae-9e23866a3b7c.png#clientId=uf44feb28-a9d2-4&from=paste&height=680&id=u47c05aaa&originHeight=680&originWidth=674&originalType=binary&ratio=1&size=15561&status=done&style=none&taskId=ua56060dc-4a23-41e2-b435-14fbd6ee4aa&width=674)

![image-20211108152607132.png](https://cdn.nlark.com/yuque/0/2021/png/12460206/1637141791449-9c68127a-7f1e-455d-b6bb-aca15c181009.png#clientId=uf44feb28-a9d2-4&from=paste&height=796&id=ueef96390&originHeight=796&originWidth=1799&originalType=binary&ratio=1&size=952132&status=done&style=none&taskId=ub001e58a-3203-46ce-885a-71204ec1502&width=1799)

复制这一段代码，到`Bootstrap`类中。

![image-20211108152814651.png](https://cdn.nlark.com/yuque/0/2021/png/12460206/1637141796873-8fdc369f-1369-4407-98e7-9da4b29c75a7.png#clientId=uf44feb28-a9d2-4&from=paste&height=594&id=u2c63dc78&originHeight=594&originWidth=1330&originalType=binary&ratio=1&size=549572&status=done&style=none&taskId=u7f3e0ed0-407c-4b41-a75e-409a58d0645&width=1330)

再次启动`Bootstrap`类，访问`localhost:8080`, 出现以下界面：

![image-20211108153001551.png](https://cdn.nlark.com/yuque/0/2021/png/12460206/1637141802307-65f41964-0e03-4e9d-8996-7e098ce16782.png#clientId=uf44feb28-a9d2-4&from=paste&height=951&id=u43ba3a58&originHeight=951&originWidth=1254&originalType=binary&ratio=1&size=163343&status=done&style=none&taskId=u199e7300-2476-49b5-89d3-49a13d443f6&width=1254)

至此，`tomcat 9.0.53`源码阅读调试环境，搭建完毕。
