---
layout: post
title: 编译优化--修改DeriveData 路径方案调研
categories: blog
description: 编译优化
keywords: docker，编译优化
---

一、背景需求

由于XX项目编译时间太长（编译一次大概需要70分钟），出现线上问题后，无法迅速排查问题，业务组同学提出了需求，希望将排查问题时，编译时间缩短，经过讨论，就有了以下方案

二、方案

每次发版时，将编译结果保存在文件服务器上。发生线上问题时，拉去对应代码，将derivedata  path 指向文件服务器driverdata 路径。利用发版时保存的编译结果，来加快编译速度。这个方案有以下几个问题：

1、DeriveData path指向文件服务器时，必须保证编译通过


2、编译结果复用的问题，否则没有意义


3、编译结果保护的问题

针对以上这几个问题，大概解决思路是，在文件服务器上开启docker ，在docker中开启samba 服务，将samba共享目录挂载到本地，然后将DeriveData path修改为挂载目录，这里可能牵涉到docker 容器内，宿主机器（文件服务器），Xcode客户端主机，三端



三、调研过程

<b>使用docker 提供samba 服务</b>

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
以上参数说明参考 [dperson/samba](https://github.com/dperson/samba)

查看docker 状态

```
$ docker ps -a

CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS                        PORTS                                                                                                                                            NAMES
7a53eb2cdb43        dperson/samba       "/sbin/tini -- /usr/…"   About a minute ago   Up About a minute (healthy)   0.0.0.0:32836->131/tcp, 0.0.0.0:32801->137/udp, 0.0.0.0:32800->138/udp, 0.0.0.0:32835->139/tcp, 0.0.0.0:32834->444/tcp, 0.0.0.0:32833->445/tcp   haibing_samba_share01

```

```
$ docker inspect 6bf88c1acf30   //查看更加详细的信息
```

3、将共享目录挂载到xcode 主机

a. 使用图形化


b. 使用如下命令

```
$ mount_smbfs -f 0777 -d 0777 smb://test01:asdf@10.235.11.29:32843/test01 /Users/haibing6/docker/
$ df
Filesystem                         512-blocks      Used Available Capacity   iused      ifree %iused  Mounted on
/dev/disk1s1                        489620264  21102160  94537688    19%    483956 2447617364    0%   /
devfs                                     382       382         0   100%       662          0  100%   /dev
/dev/disk1s2                        489620264 353626696  94537688    79%   2407025 2445694295    0%   /System/Volumes/Data
/dev/disk1s5                        489620264  18874520  94537688    17%         9 2448101311    0%   /private/var/vm
map -fstab                                  0         0         0   100%         0          0  100%   /System/Volumes/Data/Network/Servers
//jungao@10.235.24.27/share         489620264 286403792 203216472    59%  35800472   25402059   58%   /Volumes/share
//test01@10.235.11.29:32843/test01  959337808 548458560 410879248    58% 274229278  205439624   57%   /Users/haibing6/docker
```

4、将xcode DeriverData 目录设置为 `/Users/haibing6/docker` 

5、在docker 内将目录权限设置为 0777，liunx 真机设置为 0777，xcode 主机设置为 挂载时将权限设置为 0777，如果使用图形化挂载，无法更改权限

开始 build 



<b>使用linux 真机提供samba 服务</b>

```
Couldn't create workspace arena folder '/Volumes/test01/Developer/Xcode/DerivedData/Weibo-
aieyofptoenervabpoivclypymrr': Unable to write to info file '<DVTFilePath:0x7fce04f551d0:'/
Volumes/test01/Developer/Xcode/DerivedData/Weibo-aieyofptoenervabpoivclypymrr/info.plist'>'.
```



```
error: failed writing unit data: failed to rename '/Volumes/test01/Developer/Xcode/DerivedData/
Weibo-aieyofptoenervabpoivclypymrr/Index/DataStore/v5/UIKit-1V5UHAPTOD24G.pcm-32VHFZ4YQ9L0N-
bc1ed27a' to '/Volumes/test01/Developer/Xcode/DerivedData/Weibo-aieyofptoenervabpoivclypymrr/
Index/DataStore/v5/units/UIKit-1V5UHAPTOD24G.pcm-32VHFZ4YQ9L0N': File exists
1 error generated.
In file included from /Users/haibing6/work/weibomain/WeiboMain2/WeiboMain/Weibo/
StickerExtension/WBStickerBrowserViewController.m:9:
/Users/haibing6/work/weibomain/WeiboMain2/WeiboMain/Weibo/StickerExtension/
WBStickerBrowserViewController.h:9:9: fatal error: could not build module 'UIKit'
#import <UIKit/UIKit.h>
 ~~~~~~~^
2 errors generated.
```

对于以上错误，将UIKit.framework 添加到 `Link Binary With Libraries` 中，添加 `New Run Script Phase`，
内容：

```
rm -rf /Volumes/test01/Developer/Xcode/DerivedData/Weibo-aieyofptoenervabpoivclypymrr/Ind
ex/DataStore/*
```

重新编译仍然报`failed to rename `这个错误


```
In short, Cocoapods and New build system don’t work well together.

Conclusion
The new build system from Apple is designed to improve the performance, stability and 
reliability of the Swift build. It will catch the configuration errors early in the application
development. It’s been activated by default in Xcode 10 so sooner or later we have to update 
our build process to adapt to the new build system. 

```

尝试着将`new build system` 改为 `legacy build system`

再次编译，仍然报原来的错误


接着进行一下尝试：


在xcode主机上编译成功后，将DeriveData 拷贝到linux 主机，再次编译，仍然报原来的错误


<b>使用Mac 机器提供samba 服务</b>
