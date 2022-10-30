---
title: CentOS7 防火墙相关命令
categories:
  - Linux
tags: Linux
abbrlink: 372de7b5
date: 2020-10-02 23:30:31
---



# CentOS7 防火墙相关命令

1. 查看防火墙状态

```shell
firewall-cmd --state
```

2. 停止firewall

```shell
systemctl stop firewalld.service
```

3. 开启firewall

```shell
systemctl stop firewalld.service
```

4. 禁止firewall 开机启动

```she
systemctl disable firewalld.service
```

5. 设置开机启动防火墙

```shel
systemctl enable firewalld.service
```

6. 重启防火墙

```shell
firewall-cmd --reload
```

7. 对外开放端口

```shell
#说明:
#–zone 作用域
#–add-port=80/tcp 添加端口，格式为：端口/通讯协议
#–permanent 永久生效，没有此参数重启后失效
firewall-cmd --zone=public --add-port=端口号/tcp --permanent
```

8. 关闭对外端口

```shell
firewall-cmd --zone=public --remove-port=端口号/tcp --permanent
```

9. 查看已经对外开放的端口

```shell
# 设置开放某个端口后，需要重启防火墙，才能看到刚刚开启的端口
firewall-cmd --list-ports
```

10. 查看防火墙所有相关信息

```shell
firewall-cmd --list-all
```

