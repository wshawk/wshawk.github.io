---
title: Linux安装Redis 6.0.5 ./install_server.sh报错
categories:
  - Linux
tags:
  - Redis
  - Linux
abbrlink: d30a1526
date: 2021-05-10 23:30:31
---



# Linux安装Redis 6.0.5   ./install_server.sh报错

linux 安装Redis6.0.5时
进行到`./install_server.sh`时报错，

```shell
This systems seems to use systemd.
Please take a look at the provided example service unit files in this directory, and adapt and install them. Sorry!
```


解决方案：

```shell
vi ./install_server.sh
```

注释下面的代码即可

``` shell
#bail if this system is managed by systemd
#_pid_1_exe="$(readlink -f /proc/1/exe)"
#if [ "${_pid_1_exe##*/}" = systemd ]
#then
#       echo "This systems seems to use systemd."
#       echo "Please take a look at the provided example service unit files in this directory, and adapt and install them. Sorry!"
#       exit 1
#fi
```

然后重新运行 `./install_server.sh`即可。

