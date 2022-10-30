---
title: Linux环境工作常用命令
categories:
  - 后端
tags: Linux
abbrlink: 5b3efa13
date: 2019-10-15 19:51:31
---
### Linux环境工作常用命令  

<code>  
cd /  进入根目录  
mkdir dirName  创建文件夹    
touch fileName  创建一个空文件
vi/vim fileName 编辑一个文件，如果文件不存在，则会新建该文件  
mv fileName/dirName 剪切/修改 文件或者文件夹的名称  
cp -r sourceDirPath targetDirPath  复制文件夹，会将子文件夹一并复制  
tail -numf fileName  查看文件末尾num行，可以动态刷新文件，用于查看日志  
&& 命令连接符 可以多条命令连接起来，从左到右执行
ps -ef|grep processName  查看某个进程的状态
kill -9 processId 根据进程ID杀进程  
rm -rf fileName/dirName/dirPath 强制删除某个文件或者文件夹下所有内容，请慎用。如果不要强制，则去掉f  
ssh userName@HostIP  ssh远程连接  
ls -l | grep "^-" | wc -l 查看文件夹下文件个数（不包括子文件夹）  
scp -r  local_file remote_username@remote_ip:remote_folder  从本地复制到远程（从远程复制到本地，只需要将两个参数的位置调换一下）  
解压命令  
tar -xvf file.tar //解压 tar包  
tar -xzvf file.tar.gz //解压tar.gz  
tar -xjvf file.tar.bz2   //解压 tar.bz2  
tar -xZvf file.tar.Z   //解压tar.Z  
unrar e file.rar //解压rar  
unzip file.zip //解压zip
</code>