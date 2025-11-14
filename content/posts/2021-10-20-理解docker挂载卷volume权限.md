---
title: 理解Docker挂载卷文件权限
date: 2021-10-20T07:07:12+00:00
categories:
  - Docker
  - 编程
tags:
  - docker
  - volume
---

## 一、默认挂载权限问题

首先，本地挂载点新建测试文件test.txt，然后将该目录挂载到容器的test目录

![](https://objects.wangaiguo.com/wp-content/uploads/2021/10/image-7.png)

可以看到我们在容器外新建的test.txt文件在容器内的所有者变成了1000，然后我们在容器内新建文件查看容器外的权限，发现容器中新建的test1.txt所有者变成了root

![](https://objects.wangaiguo.com/wp-content/uploads/2021/10/image-8.png)

接着查看下UID：1000所属用户

![](https://objects.wangaiguo.com/wp-content/uploads/2021/10/image-9.png)

最后我们在指定wag用户启动docker，然后在容器内新建test3.txt文件，查看文件所有者

![](https://objects.wangaiguo.com/wp-content/uploads/2021/10/image-10.png)

## 二、结论

从以上可以得出，docker启动容器如果不指定用户，会默认以**root（UID=0）**方式运行，导致其中新建的文件所有者映射到容器外为root，容器外新建的文件映射到容器内所有者UID不变。

### 容器共享宿主机的UID

Linux内核负责管理uid和gid，并通过内核级别的系统调用来决定是否通过请求的权限。比如，当一个进程尝试去写文件，内核会检查创建这个进程的的user的uid和gid，来决定这个进程是否有权限修改这个文件。这里没有使用username，而是uid。

当docker容器运行在宿主机上的时候，仍然只有一个内核。容器共享宿主机的内核，所以所有的uid和gid都受同一个内核来控制.
