---
layout: post
title: 编译优化--修改DeriveData 路径方案调研
categories: blog
description: 编译优化
keywords: docker，编译优化
---

一、背景需求

由于微博项目编译时间太长（编译一次大概需要70分钟），出现线上问题后，无法迅速排查问题，业务组同学提出了需求，希望将排查问题时，编译时间缩短，经过讨论，就有了以下方案

二、方案

每次发版时，将编译结果保存在文件服务器上。发生线上问题时，拉去对应代码，将derivedata  path 指向文件服务器driverdata 路径。利用发版时保存的编译结果，来加快编译速度。这个方案有以下几个问题：

1、DeriveData path指向文件服务器时，必须保证编译通过


2、编译结果复用的问题，否则没有意义


3、编译结果保护的问题

针对以上这几个问题，大概解决思路是，在文件服务器上开启docker ，在docker中开启samba 服务，将samba共享目录挂载到本地，然后将DeriveData path修改为挂载目录，这里可能牵涉到docker 容器内，宿主机器（文件服务器），Xcode客户端主机，三端



三、调研过程



1、docker 搭建，参考 [Docker 教程](https://www.runoob.com/docker/docker-tutorial.html)

2、安装镜像


a. 查找镜像
 
```
$ docker search samba
NAME                              DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
dperson/samba                                                                     385                                     [OK]
svendowideit/samba                Sharing a Docker container's volume should b…   55                                      [OK]
nowsci/samba-domain               A well documented and tested Samba Active Di…   24                                      [OK]
servercontainers/samba            samba - complied from official stable releas…   17                                      [OK]
appcontainers/samba               CentOS 6.6 Samba 4 Container - 282.2MB          13                                      [OK]
elswork/samba                     Multi-Arch container of Samba for AMD & ARM …   12                                      
jenserat/samba-publicshare        Simple Docker image for publically sharing a…   12                                      [OK]
joebiellik/samba-server           Simple Samba server running on Alpine Linux …   9                                       [OK]
sonohara/samba4-ad                Samba 4 ActiveDirectory docker                  9                                       [OK]
dreamcat4/samba                                                                   8                                       [OK]
pwntr/samba-alpine                Simple and lightweight Samba docker containe…   6                                       [OK]
gists/samba-server                Samba server based on alpine                    5                                       [OK]
rsippl/samba-ad-dc                Samba 4 Active Directory Domain Controller      4                                       [OK]
timjdfletcher/samba-timemachine   Samba configured to run as a timemachine tar…   4                                       
sixeyed/samba                     Samba server, FROM dperson/samba                3                                       [OK]
andrespp/samba-ldap               Docker image for SAMBA with LDAP authenticat…   3                                       [OK]
```
 
采用下载次数最多的镜像`dperson/samba`


b. 拉取镜像

```
$docker pull  dperson/samba
```


c. 在本地创建一个目录，用于与docker 的目录映射


```
$ mkdir /home/shared
$ chmod 777 /home/shared  //修改shared权限，不修改的话连接进去会提示没有权限写入数据
```


d. 启动镜像

```
$ docker run -it --name haibing_samba_share01 -P \
> -v /home/test01/docker_share01:/test01 \
> -d dperson/samba \
> -u "test01;asdf" \
> -s "haibing_samba_share01;/test01;yes;no;yes;all;none"

7a53eb2cdb431abad1ebd5214f3fabb868ce8c8e1dd40bd33f4a2fa55f19be03
```
如果出现一串字符，docker 的标识符，表示docker 创建成功
以上参数说明参考[dperson/samba](https://github.com/dperson/samba)

查看docker 状态

```
$ docker ps -a

CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS                        PORTS                                                                                                                                            NAMES
7a53eb2cdb43        dperson/samba       "/sbin/tini -- /usr/…"   About a minute ago   Up About a minute (healthy)   0.0.0.0:32836->131/tcp, 0.0.0.0:32801->137/udp, 0.0.0.0:32800->138/udp, 0.0.0.0:32835->139/tcp, 0.0.0.0:32834->444/tcp, 0.0.0.0:32833->445/tcp   haibing_samba_share01

```


```
Couldn't create workspace arena folder '/Volumes/test01/Developer/Xcode/DerivedData/Weibo-aieyofptoenervabpoivclypymrr': Unable to write to info file '<DVTFilePath:0x7fce04f551d0:'/Volumes/test01/Developer/Xcode/DerivedData/Weibo-aieyofptoenervabpoivclypymrr/info.plist'>'.
```
